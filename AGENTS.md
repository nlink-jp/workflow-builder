# AGENTS.md — workflow-builder

LLM-powered workflow builder for nlink-jp CLI tool ecosystem.
Part of [lab-series](https://github.com/nlink-jp/lab-series).

**Status: Design phase — no runnable code yet.**

## Key structure

```
tools/*.yaml             ← Tool registry definitions
docs/design/             ← Architecture document
container/Dockerfile     ← Toolbox container skeleton
```

## Design

- Tool registry: YAML with name, description, flags, examples, pipes_well_with
- Orchestrator: Plan (CoT) → Generate (script) → Execute (container) → Evaluate → Iterate
- Container: podman/docker sandbox with nlink-jp tools + jq/curl/etc

See [docs/design/architecture.md](docs/design/architecture.md) for full details.
