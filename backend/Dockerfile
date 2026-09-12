# Backend Dockerfile - multi-stage build
# Stage 1 (builder): compiles native Python extensions with build-essential
# Stage 2 (dev):     retains toolchain for dev containers (uv sync at startup)
# Stage 3 (runtime): clean image without compiler toolchain for production

# Source images (override for restricted networks if needed)
ARG UV_IMAGE=ghcr.io/astral-sh/uv:0.11.1
ARG NODE_MAJOR=22
FROM ${UV_IMAGE} AS uv-source
FROM node:${NODE_MAJOR}-bookworm-slim AS node-source

# -- Stage 1: Builder ----------------------------------------------------------
FROM python:3.12-slim-bookworm AS builder

ARG APT_MIRROR
ARG UV_INDEX_URL
ARG NPM_REGISTRY
ARG LARK_CLI_NPM_VERSION=1.0.65
# Optional extras to install (e.g. "postgres" for PostgreSQL support)
# Usage: docker build --build-arg UV_EXTRAS=postgres,discord ...
ARG UV_EXTRAS

# Optionally override apt mirror for restricted networks (e.g. APT_MIRROR=mirrors.aliyun.com)
RUN if [ -n "${APT_MIRROR}" ]; then \
      sed -i "s|deb.debian.org|${APT_MIRROR}|g" /etc/apt/sources.list.d/debian.sources 2>/dev/null || true; \
      sed -i "s|deb.debian.org|${APT_MIRROR}|g" /etc/apt/sources.list 2>/dev/null || true; \
    fi

# Install build tools (build-essential needed for native Python extensions)
RUN apt-get update && apt-get install -y \
    curl \
    git \
    build-essential \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Provide Node.js + npm without using NodeSource bootstrap (helps restricted TLS networks).
COPY --from=node-source /usr/local/ /usr/local/

# Install the official Lark/Feishu CLI used by the managed Lark integration.
RUN if [ -n "${NPM_REGISTRY}" ]; then npm config set registry "${NPM_REGISTRY}"; fi \
    && npm install -g @larksuite/cli@${LARK_CLI_NPM_VERSION} \
    && lark-cli --version

# Install uv (source image overridable via UV_IMAGE build arg)
COPY --from=uv-source /uv /uvx /usr/local/bin/

# Set working directory
WORKDIR /app

# Copy backend source code
COPY backend ./backend

# Install dependencies with cache mount
# `redis` is always installed because Docker Compose defaults the stream bridge
# to Redis (DEER_FLOW_STREAM_BRIDGE_REDIS_URL); it is an optional extra elsewhere
# so slim non-Docker installs do not pull it.
# When UV_EXTRAS is set (comma- or whitespace-separated), installs optional dependencies.
RUN --mount=type=cache,target=/root/.cache/uv \
    sh -c 'cd backend && \
    set -f; \
    extras_flags=""; \
    for extra in $(printf "%s" "$UV_EXTRAS" | tr "," " "); do \
        case "$extra" in \
            [!A-Za-z]* | *[!A-Za-z0-9_-]*) \
                echo "UV_EXTRAS entry $extra is invalid (must match [A-Za-z][A-Za-z0-9_-]*)" >&2; \
                exit 1; \
                ;; \
        esac; \
        extras_flags="$extras_flags --extra $extra"; \
    done; \
    UV_INDEX_URL=${UV_INDEX_URL:-https://pypi.org/simple} uv sync --locked --extra redis $extras_flags'

# UTF-8 locale prevents UnicodeEncodeError on Chinese/emoji content in minimal
# containers where locale configuration may be missing and the default encoding is not UTF-8.
ENV LANG=C.UTF-8
ENV LC_ALL=C.UTF-8
ENV PYTHONIOENCODING=utf-8

# -- Stage 2: Dev --------------------------------------------------------------
# Retains compiler toolchain from builder so startup-time `uv sync` can build
# source distributions in development containers.
FROM builder AS dev

# Install Docker CLI (for DooD: allows starting sandbox containers via host Docker socket)
COPY --from=docker:cli /usr/local/bin/docker /usr/local/bin/docker

EXPOSE 8001

CMD ["sh", "-c", "cd backend && PYTHONPATH=. uv run --no-sync uvicorn app.gateway.app:app --host 0.0.0.0 --port 8001"]

# -- Stage 3: Runtime ----------------------------------------------------------
# Clean image without build-essential - reduces size and attack surface.
FROM python:3.12-slim-bookworm

ENV LANG=C.UTF-8
ENV LC_ALL=C.UTF-8
ENV PYTHONIOENCODING=utf-8

# Copy Node.js runtime from builder (provides npm/npx and global CLI tools for MCP servers)
COPY --from=builder /usr/local/bin/node /usr/local/bin/node
COPY --from=builder /usr/local/bin/npm /usr/local/bin/npm
COPY --from=builder /usr/local/bin/npx /usr/local/bin/npx
COPY --from=builder /usr/local/bin/lark-cli /usr/local/bin/lark-cli
COPY --from=builder /usr/local/lib/node_modules /usr/local/lib/node_modules
RUN lark-cli --version

# Install Docker CLI (for DooD: allows starting sandbox containers via host Docker socket)
COPY --from=docker:cli /usr/local/bin/docker /usr/local/bin/docker

# Install uv (source image overridable via UV_IMAGE build arg)
COPY --from=uv-source /uv /uvx /usr/local/bin/

# Set working directory
WORKDIR /app

# Copy backend with pre-built virtualenv from builder
COPY --from=builder /app/backend ./backend

# Expose Gateway API port.
EXPOSE 8001

# Default command (can be overridden in docker-compose)
CMD ["sh", "-c", "cd backend && PYTHONPATH=. uv run --no-sync uvicorn app.gateway.app:app --host 0.0.0.0 --port 8001"]
