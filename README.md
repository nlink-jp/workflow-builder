# workflow-builder

An LLM-powered workflow builder that generates executable shell scripts from natural language task descriptions. Uses a structured tool registry to understand available CLI tools, plans data flows, and iteratively refines scripts through containerized execution and feedback.

**Status: Design phase** — architecture and tool registry defined; implementation pending.

[日本語版 README はこちら](README.ja.md)

## Concept

```
"Collect yesterday's cybersecurity news,    ──→  LLM analyzes task
 translate to Japanese, and post             ──→  Plans tool chain
 a curated digest to #security-news"         ──→  Generates shell script
                                             ──→  Executes in container
                                             ──→  Validates result
                                             ──→  Outputs working workflow
```

## Components

| Component | Description |
|---|---|
| **Tool Registry** (`tools/*.yaml`) | Structured definitions of CLI tools — flags, I/O formats, examples, composition hints |
| **Container Image** (`container/`) | Sandboxed execution environment with nlink-jp tools + standard utilities pre-installed |
| **Orchestrator** | LLM-driven pipeline: Plan → Generate → Execute → Evaluate → Iterate |

## How It Works

1. **Define** available tools in YAML (what they do, how to use them)
2. **Describe** a task in natural language
3. **Plan** — LLM reads tool registry + task, produces Chain-of-Thought reasoning
4. **Generate** — LLM converts the plan into a shell script
5. **Execute** — Script runs inside a container (sandboxed)
6. **Evaluate** — LLM analyzes results, determines success or diagnoses errors
7. **Iterate** — If needed, regenerate with fixes (max N attempts)
8. **Output** — Working shell script, ready for reuse

## Tool Registry

Tools are defined in `tools/*.yaml` with structured metadata:

```yaml
name: gem-cli
description: Gemini CLI client for Vertex AI
flags:
  - name: --format
    type: enum
    values: [text, json, jsonl]
examples:
  - description: Extract structured data
    command: echo "$DATA" | gem-cli -s "Extract fields" --format json
pipes_well_with: [jq, swrite, json-to-table]
```

Currently registered tools: gem-cli, swrite, news-collector (more to come).

## Architecture

See [docs/design/architecture.md](docs/design/architecture.md) for the full design document.

## Documentation

- [Architecture](docs/design/architecture.md) — components, data flow, security, design decisions
