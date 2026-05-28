# syntax=docker/dockerfile:1.7
# Production-grade Python 3.13 Dockerfile
# Multi-stage, uv-based, distroless-friendly, non-root, BuildKit cache mounts

ARG PYTHON_VERSION=3.13
ARG UV_VERSION=0.5.4

# ─── Stage 1: builder ────────────────────────────────────────────────
FROM python:${PYTHON_VERSION}-slim-bookworm AS builder

COPY --from=ghcr.io/astral-sh/uv:${UV_VERSION} /uv /uvx /bin/

ENV UV_LINK_MODE=copy \
    UV_COMPILE_BYTECODE=1 \
    UV_PYTHON_DOWNLOADS=never \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends \
        build-essential

COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-install-project --no-dev

COPY . .
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev

# ─── Stage 2: runtime ────────────────────────────────────────────────
FROM python:${PYTHON_VERSION}-slim-bookworm AS runtime

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends \
        curl ca-certificates \
 && rm -rf /var/lib/apt/lists/*

RUN groupadd -r app --gid 10001 \
 && useradd -r -g app --uid 10001 --home /app --shell /sbin/nologin app

WORKDIR /app

COPY --from=builder --chown=app:app /app/.venv /app/.venv
COPY --from=builder --chown=app:app /app /app

ENV PATH="/app/.venv/bin:$PATH" \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PYTHONPATH=/app/src

USER 10001:10001

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD curl -fsS http://localhost:8000/health || exit 1

LABEL org.opencontainers.image.source="https://github.com/justxor/Docker-Roamap-2026"
LABEL org.opencontainers.image.description="Docker Roadmap 2026 — starter template"
LABEL org.opencontainers.image.licenses="MIT"

ENTRYPOINT ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
