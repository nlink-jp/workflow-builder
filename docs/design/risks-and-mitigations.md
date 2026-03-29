# Known Risks and Mitigations

This document catalogues identified risks from design reviews and outlines
mitigation strategies. These are deliberate trade-offs to be validated
during the prototyping phase.

---

## 1. Build Cost and Latency

**Risk:** Each workflow compilation requires multiple LLM API calls (plan,
generate, execute, evaluate, iterate). Complex pipelines may require many
iterations, making the "build" phase expensive and slow.

**Concern:** For frequently-changing tasks, the build cost may approach
the cost of just running an LLM agent directly.

**Mitigations:**

- Use **Gemini Flash** (low cost) for routine generation; reserve Pro for
  complex planning only
- **Cache successful patterns**: if a similar task was compiled before,
  reuse the previous script as a starting point
- **Incremental compilation**: when a task changes slightly, modify the
  existing script rather than regenerating from scratch
- **Cost tracking**: log API token usage per compilation to identify
  expensive patterns

**Acceptance:** This is a real trade-off. The value proposition holds when
workflows run many times (daily cron) — the one-time build cost is
amortized. For one-shot tasks, direct LLM execution may be more practical.

---

## 2. Shell Script Limitations and Maintainability

**Risk:** Bash is weak at complex data structures, error handling, and
readability. LLM-generated scripts may "work" but be hard for humans to
maintain. Silent data loss in pipe chains is hard to detect.

**Mitigations:**

- **Pipeline validation**: instruct the LLM to add intermediate
  `wc -l` / `head` checks and stderr logging at each pipe stage
- **Generated comments**: require the LLM to comment every non-trivial
  line explaining intent
- **Shellcheck integration**: run `shellcheck` on generated scripts as
  a post-generation quality gate
- **Re-compilation over maintenance**: treat generated scripts as
  disposable artifacts. When the task changes, re-compile rather than
  hand-edit. Version the task description, not the script.

**Idempotency enforcement:**

Rather than relying solely on LLM prompts to produce idempotent scripts,
the orchestrator injects structural idempotency patterns at code generation
time. The tool registry provides per-tool idempotency hints:

```yaml
# In tool registry
idempotency:
  strategy: dedup-by-id    # or: check-before-write, atomic-replace, append-safe
  existing_output: skip    # skip | overwrite | error
```

The code generator uses these hints to wrap tool invocations:

```bash
# Auto-injected: check-before-write pattern
if [ ! -f /workspace/output/report.json ]; then
  gem-cli "Analyze this" --image input.png --format json     > /workspace/output/report.json
else
  echo "[skip] report.json already exists" >&2
fi
```

This removes idempotency from "things the LLM might forget" and makes it
a structural guarantee of the code generator.



A reviewer suggested generating a YAML/JSON workflow definition first
(similar to GitHub Actions), then transpiling to Bash deterministically.
This is worth exploring:

```
Task (NL) → LLM → Workflow IR (YAML) → Deterministic transpiler → Bash
```

Pros:
- LLM generates structured data (easier to validate) instead of raw code
- Transpiler handles error trapping, cleanup, logging uniformly
- IR is more auditable than raw Bash

Cons:
- Additional complexity (need to design IR schema + transpiler)
- May limit expressiveness for edge cases
- Two layers to debug instead of one

**Decision:** Start with direct Bash generation for the prototype. If
script quality proves problematic, introduce IR as a v2 architecture.
The tool registry already provides structured data that could feed
an IR design.

---

## 3. Tool Registry Maintenance Burden

**Risk:** Manual YAML maintenance doesn't scale. Tool updates (new flags,
breaking changes) require manual registry edits. Data-level dependencies
between tools (e.g. "tool A outputs JSON with key X that tool B requires")
are hard to express in YAML.

**Mitigations:**

- **Auto-generation from `--help`**: planned feature to generate draft
  YAML from a tool's help output (see architecture.md)
- **Version pinning**: registry entries include tool version; CI can
  detect mismatches between installed version and registry version
- **Schema validation**: define a JSON Schema for tool registry YAML
  and validate on commit
- **Data contract hints**: extend the registry with `output_schema`
  and `input_schema` fields for tools that produce/consume JSON:

  ```yaml
  output_schema:
    format: jsonl
    fields:
      - name: title
        type: string
      - name: tags
        type: string[]
  ```

  This lets the orchestrator validate pipe compatibility at plan time,
  before generating any code.

- **Static pipe validation**: at plan time, before any code is generated,
  the orchestrator checks that each pipe stage's `output_schema` is
  compatible with the next stage's `input_schema`. Mismatches are flagged
  immediately:

  ```
  [PLAN ERROR] news-collector curate outputs JSONL with {blocks: [...]},
  but swrite --format blocks expects a bare JSON array.
  Suggested fix: pipe through 'jq .blocks[]'
  ```

  This eliminates the most common class of "works in isolation but breaks
  in the pipeline" errors — without executing anything.

- **Contract versioning**: schemas include the tool version they describe.
  When a tool is updated, the orchestrator can detect outdated contracts
  and prompt for re-validation.

---

## 4. Container Image Size and Dependency Conflicts

**Risk:** Bundling all tools into one image leads to multi-GB images and
potential dependency conflicts (especially Python tools).

**Mitigations:**

- **Multi-stage builds**: compile Go binaries in a builder stage, copy
  only the static binaries to the final slim image
- **Python isolation**: use `uv tool install` which creates isolated
  virtualenvs per tool (no cross-tool dependency conflicts)
- **Layered images**: consider a base image with standard utilities,
  and tool-specific layers that can be composed. Or multiple images
  (Go-tools, Python-tools) selected based on the workflow's needs.
- **Image size budget**: set a target (e.g. < 1 GB) and track it in CI

**Note:** Go tools are static binaries (< 10 MB each). The primary size
contributor is the Python runtime + tool dependencies. `uv` helps by
avoiding pip's dependency resolution issues.

---

## 5. Security: Script Injection

**Risk:** A malicious or careless task description could cause the LLM
to generate destructive commands (`rm -rf`, credential exfiltration).

**Mitigations:**

- **Container sandboxing**: destructive commands affect only the
  container, not the host
- **Read-only input mount**: `/workspace/input` mounted as read-only
- **Static analysis**: run generated scripts through a linter/validator
  that flags dangerous patterns:
  - `rm -rf` outside `/workspace/tmp`
  - `curl` to unexpected domains
  - Credential echo/logging
  - Unbounded loops
- **Blocklist**: the orchestrator rejects scripts containing known
  dangerous patterns before execution
- **Network allowlist**: when network is enabled, restrict to known
  API endpoints (Vertex AI, Slack API) via container networking rules

---

## 6. Summary: Risk vs Phase

| Risk | Severity | Phase to Address |
|---|---|---|
| Build cost/latency | Medium | Prototype (measure actual costs) |
| Bash limitations | Medium | Prototype → v2 (IR if needed) |
| Registry maintenance | Low (initially) | Scale-up phase |
| Container size | Low | Scale-up phase |
| Script injection | High | Prototype (must-have from day 1) |
