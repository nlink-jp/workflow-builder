# Architecture — workflow-builder

## Overview

workflow-builder is an LLM-powered system that generates executable shell
scripts from natural language task descriptions. It uses a structured tool
registry to understand available CLI tools, plans data flows between them,
and iteratively refines the generated script through execution feedback.

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

- Generated scripts run **inside a container**, not on the host
- Container has **no network access by default** (opt-in via flag)
- Input/output via **mounted volumes** only
- API credentials passed via **environment variables** (not baked into image)
- Scripts are **logged and reviewable** before execution (dry-run mode)
- Execution has a **configurable timeout** (default: 5 minutes)

## Future Directions

- **Workflow library**: save and share proven workflows
- **Parameterized workflows**: templates with variable substitution
- **Scheduled execution**: cron-like recurring workflow runs
- **Multi-step approval**: human-in-the-loop before execution
- **Tool auto-discovery**: scan installed binaries and generate registry entries
