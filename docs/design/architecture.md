# Architecture — workflow-builder

## Core Thesis

LLM as an autonomous agent is **probabilistic** — the same prompt may
produce different outputs on each run, and those outputs may or may not
be correct. This makes LLM-driven task execution unreliable when used
directly (e.g. "run this agent every day at 07:00").

workflow-builder addresses this by **separating the probabilistic phase
from the deterministic phase:**

```
Probabilistic Phase                 Deterministic Phase
(LLM, human-in-the-loop)           (shell script, cron, CI)

  Task description                    Compiled workflow
  + Tool registry                     = shell script
  + CoT planning          ──compile──→   that runs the same way
  + Script generation                    every time,
  + Execution feedback                   with no LLM involvement
  + Human approval                       at runtime.
```

The LLM is used **at build time** (design, planning, code generation,
debugging) but is **absent at run time**. The output is a conventional
shell script — fully deterministic, auditable, version-controllable,
and executable without any AI dependency.

This is analogous to how a compiler transforms high-level source code
into machine code: the compilation step may involve complex heuristics,
but the resulting binary runs predictably.

**In short: use the LLM to compile workflows, not to execute them.**

---

## Overview

workflow-builder is a system that uses an LLM to generate executable
shell scripts from natural language task descriptions. It uses a structured
tool registry to understand available CLI tools, plans data flows between
them, and iteratively refines the generated script through containerized
execution and human feedback — until the workflow is proven correct and
ready for autonomous, LLM-free operation.

## Components

```
┌─────────────────────────────────────────────────────────────┐
│                     workflow-builder                         │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Tool Registry │  │  Orchestrator │  │ Container Image  │  │
│  │              │  │              │  │                  │  │
│  │ tools/*.yaml │→│ LLM Planner  │→│ Sandboxed exec   │  │
│  │              │  │ LLM Coder   │  │ nlink-jp tools   │  │
│  │ CLI specs    │  │ Feedback    │  │ + jq, curl, etc  │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1. Tool Registry (`tools/*.yaml`)

Structured YAML definitions of available CLI tools. Each definition includes:

- **Name and description** — what the tool does
- **Input/output spec** — stdin/stdout formats, file types
- **Flags and parameters** — with types, defaults, descriptions
- **Usage examples** — concrete command lines for common patterns
- **Composition hints** — which tools it pipes well with

These definitions serve as the LLM's "knowledge base" about available tools.
The LLM uses them to plan data flows and generate correct command invocations.

### 2. Container Image (`container/`)

A pre-built container image with all registered tools installed:

- **nlink-jp tools**: gem-cli, swrite, stail, news-collector, json-to-table,
  rex, sdate, csv-to-json, json-filter, lookup, jstats, jviz, etc.
- **Standard utilities**: jq, curl, grep, sed, awk, python3
- **Runtime**: bash as the shell interpreter

The container provides:
- **Sandboxed execution** — generated scripts run in isolation
- **Reproducibility** — same tools, same versions, every time
- **Security** — no access to host filesystem or network by default
  (mount volumes explicitly for input/output)

Built and managed via Dockerfile + Makefile. Podman or Docker as runtime.

### 3. Orchestrator

The core logic that ties everything together:

```
Input:  Natural language task description
        + Tool registry (YAML files)
        + Optional: input data files

Step 1: PLAN (LLM)
        - Analyze the task
        - Select relevant tools from registry
        - Design the data flow (pipe chain)
        - Produce Chain-of-Thought reasoning

Step 2: GENERATE (LLM)
        - Convert CoT plan into a shell script
        - Include error handling and validation
        - Add comments explaining each step

Step 3: EXECUTE
        - Run the script inside the container
        - Capture stdout, stderr, exit code
        - Time-bound execution (timeout)

Step 4: EVALUATE (LLM)
        - Analyze execution results
        - Determine: success, partial success, or failure
        - If failure: diagnose the issue, suggest fixes

Step 5: ITERATE (if needed)
        - Feed error analysis back to Step 2
        - Regenerate with fixes applied
        - Max N iterations (configurable)

Output: Working shell script
        + Execution log
        + Final result
```

## Data Flow Examples

### Example 1: "Collect cybersecurity news and post a digest to Slack"

```
Plan:
  1. news-collector collect --genre cybersecurity
  2. news-collector process --topics topics.toml
  3. news-collector curate --lang ja
  4. pipe each line to swrite post --format blocks --no-unfurl -c "#news"

Generated script:
  #!/bin/bash
  set -euo pipefail
  news-collector collect --topics /workspace/topics.toml --db /workspace/news.db
  news-collector process --topics /workspace/topics.toml --db /workspace/news.db
  news-collector curate --db /workspace/news.db --lang ja \
    | while IFS= read -r line; do
        printf '%s' "$line" | swrite post --format blocks --no-unfurl -c "#news"
        sleep 1
      done
```

### Example 2: "Extract all email addresses from these log files, deduplicate, and save as CSV"

```
Plan:
  1. cat log files
  2. rex with email regex pattern
  3. jq to extract matched field
  4. sort -u for deduplication
  5. output as CSV

Generated script:
  #!/bin/bash
  set -euo pipefail
  cat /workspace/input/*.log \
    | rex '(?P<email>[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})' \
    | jq -r '.email' \
    | sort -u \
    | sed '1i email' \
    > /workspace/output/emails.csv
```

### Example 3: "Analyze this image and create a structured report in Japanese"

```
Plan:
  1. gem-cli to analyze the image with Gemini
  2. gem-cli to translate and structure the output
  3. Save as JSON and Markdown

Generated script:
  #!/bin/bash
  set -euo pipefail
  gem-cli "Analyze this image in detail. Describe objects, text, and layout." \
    --image /workspace/input/screenshot.png \
    --format json \
    > /workspace/output/analysis.json
  gem-cli -f /workspace/output/analysis.json \
    -s "Translate to Japanese and format as a Markdown report with sections." \
    > /workspace/output/report.md
```

## Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Container runtime | Podman/Docker | Security (sandboxed), reproducibility |
| LLM for planning | gem-cli (Gemini) | Already built, Vertex AI, Grounding support |
| Script language | Bash | Universal, tools are CLI-native |
| Tool registry format | YAML | Human-readable, easy to author |
| Feedback loop | Max 3 iterations | Prevent infinite loops, manage API cost |
| Orchestrator language | Python or Go | TBD — Python for prototyping, Go for distribution |

## Security Considerations

### Container isolation

- Generated scripts run **inside a container**, not on the host
- Input/output via **mounted volumes** only (`/workspace/input`, `/workspace/output`)
- Execution has a **configurable timeout** (default: 5 minutes)

### Credential management

API credentials (Slack tokens, GCP service account keys, etc.) are
**never baked into the container image**. They are injected at container
startup via environment variables or mounted secret files:

```bash
podman run --rm \
  -e SWRITE_TOKEN="$(cat ~/.secrets/slack-token)" \
  -e GOOGLE_CLOUD_PROJECT="my-project" \
  -v ./workspace:/workspace \
  workflow-builder-toolbox /workspace/my-workflow.sh
```

The orchestrator must ensure credentials are passed only to containers
that need them, and only for the duration of execution.

### Network access control

By default, containers run with **no network access** (`--network=none`).
Tools that require external API calls (e.g. `gem-cli`, `swrite`,
`news-collector`) need network access explicitly enabled.

To support this, the tool registry includes a `requires_network` flag:

```yaml
name: gem-cli
requires_network: true    # needs Vertex AI API access

name: rex
requires_network: false   # pure local text processing
```

The orchestrator inspects all tools used in the generated script and
enables network access **only if at least one tool requires it**.
This minimizes the attack surface of scripts that only do local
data transformation.

### Script review and approval

All generated scripts support a **dry-run mode** where the script is
displayed for human review before execution. For destructive operations
(file deletion, external API writes, mass posting), the orchestrator
requires explicit human approval before proceeding.

---

## Idempotency and Reliability

### Idempotent script generation

For workflows intended for recurring execution (cron, scheduled jobs),
the orchestrator instructs the LLM to generate **idempotent scripts** —
scripts that produce the same result whether run once or multiple times:

- Use `INSERT OR IGNORE` / dedup patterns for data storage
- Check for existing output before regenerating
- Use atomic file operations (write to temp, then move)

The system prompt for the LLM coder includes idempotency requirements
when the user indicates the workflow will run on a schedule.

### Intermediate file management

Generated scripts use a structured workspace layout:

```
/workspace/
  input/          ← mounted read-only input data
  output/         ← final results (persisted)
  tmp/            ← intermediate files (cleaned up on exit)
```

The orchestrator wraps generated scripts with a cleanup trap:

```bash
trap 'rm -rf /workspace/tmp' EXIT
```

---

## Human-in-the-Loop

### Approval workflow

```
LLM generates script
        │
        ▼
   ┌─────────┐      ┌─────────────┐
   │ Dry run  │──→──│ Human review │
   │ (preview)│      │ Approve?    │
   └─────────┘      └──────┬──────┘
                        │       │
                    [approve] [reject + feedback]
                        │       │
                        ▼       ▼
                    Execute   Regenerate
                              with feedback
```

The orchestrator provides:

1. **`--dry-run`**: show the generated script without executing
2. **`--approve`**: require explicit "yes" before execution
3. **Risk classification**: tag scripts as low/medium/high risk based on
   operations used (read-only vs. write vs. delete vs. external API)

### Risk levels

| Level | Operations | Approval |
|---|---|---|
| Low | Local read, text processing, format conversion | Auto-approve |
| Medium | File creation, local DB writes | Prompt for confirmation |
| High | External API calls, file deletion, mass posting | Require explicit approval |

---

## Tool Registry Extensions

### `requires_network` flag

```yaml
requires_network: true   # tool needs external API access
```

Used by the orchestrator to determine container network policy.

### Auto-generation from `--help`

Future: a subcommand that reads a tool's `--help` output and generates
a draft `tools/*.yaml` definition:

```bash
workflow-builder registry generate --from-help "gem-cli --help"
```

This accelerates onboarding of new tools into the ecosystem.

---

## Testing and Validation

### Test fixture generation

When compiling a workflow, the LLM also generates:

1. **Sample input data** — minimal test fixtures for the workflow
2. **Expected output** — what the workflow should produce
3. **Validation script** — a companion script that checks the output

This enables automated regression testing of compiled workflows:

```bash
# Run the workflow with test fixtures
./my-workflow.sh < test-input.json > actual-output.json

# Validate
./my-workflow.test.sh actual-output.json expected-output.json
```

### Workflow versioning

Compiled workflows are stored with metadata:

```
workflows/
  news-digest/
    v1.sh              ← compiled script
    v1.meta.yaml       ← task description, tool versions, generation date
    v1.test-input/     ← test fixtures
    v1.expected-output/ ← expected results
```

---

## Future Directions

- **Workflow library**: save, share, and discover proven workflows
- **Parameterized workflows**: templates with variable substitution
  (`$CHANNEL`, `$DATE_RANGE`, etc.)
- **Scheduled execution**: cron-like recurring workflow runs with
  built-in idempotency verification
- **Multi-user approval**: role-based approval for high-risk workflows
- **Tool auto-discovery**: scan installed binaries and generate registry entries
- **Workflow composition**: combine multiple compiled workflows into
  larger pipelines
- **Execution analytics**: track success rates, execution times, and
  failure patterns across workflow runs
