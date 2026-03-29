# CLAUDE.md — workflow-builder

**Organization rules (mandatory): https://github.com/nlink-jp/.github/blob/main/CONVENTIONS.md**

## This project

LLM-powered workflow builder. Generates shell scripts from natural language
task descriptions using a structured CLI tool registry.

**Status: Design phase.** Architecture defined, tool registry format defined,
implementation pending.

## Key structure

```
tools/                   ← Tool registry (YAML definitions)
  gem-cli.yaml
  swrite.yaml
  news-collector.yaml
docs/design/
  architecture.md        ← Full architecture document
container/
  Dockerfile             ← Toolbox container image (skeleton)
```

## Design overview

1. Tool Registry: YAML files describing CLI tools (flags, I/O, examples, pipes_well_with)
2. Orchestrator: LLM reads registry + task → CoT → shell script → container exec → feedback loop
3. Container: sandboxed execution with nlink-jp tools pre-installed
