# workflow-builder toolbox container
# Pre-installs nlink-jp CLI tools + standard utilities for workflow execution.
#
# Build:
#   podman build -t workflow-builder-toolbox container/
#
# Run (example):
#   podman run --rm -v ./workspace:/workspace workflow-builder-toolbox /workspace/script.sh

FROM python:3.11-slim AS base

# ── System utilities ──
RUN apt-get update -qq && \
    apt-get install -y -q --no-install-recommends \
      curl unzip jq git bash coreutils grep sed gawk \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# ── uv (Python package manager) ──
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

# ── Go binaries from nlink-jp releases ──
# TODO: pin versions and download from GitHub Releases
# Example pattern:
#   ARG GEM_CLI_VERSION=v0.2.0
#   RUN curl -fsSL "https://github.com/nlink-jp/gem-cli/releases/download/${GEM_CLI_VERSION}/gem-cli-linux-amd64.zip" \
#         -o /tmp/gem-cli.zip && unzip /tmp/gem-cli.zip -d /tmp && \
#         cp /tmp/gem-cli-linux-amd64 /usr/local/bin/gem-cli && chmod +x /usr/local/bin/gem-cli

# ── Python tools from nlink-jp ──
# TODO: install via uv tool install
# Example:
#   RUN uv tool install git+https://github.com/nlink-jp/news-collector.git

# ── Workspace ──
WORKDIR /workspace

ENTRYPOINT ["/bin/bash"]
