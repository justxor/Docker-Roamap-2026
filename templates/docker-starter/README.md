# 🐳 docker-starter

Минимальный production-ready Dockerfile для Python 3.13 + uv.

## Что внутри

- Multi-stage build (builder + runtime)
- uv для зависимостей (быстрее pip в 10-100x)
- Non-root user (UID 10001)
- BuildKit cache mounts (apt, uv)
- Healthcheck
- OCI labels
- ARG для версий

## Использование

```bash
# Build
docker build -t myapp:dev .

# Multi-arch build & push
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/user/myapp:v1.0.0 \
  --push .

# Run
docker run --rm -p 8000:8000 myapp:dev
```

## Что нужно адаптировать

1. `pyproject.toml` и `uv.lock` — твои зависимости
2. `ENTRYPOINT` — стартовая команда (FastAPI/Flask/CLI)
3. `HEALTHCHECK` — путь и порт твоего `/health` endpoint
4. `PYTHONPATH` — если структура отличается от `src/`
5. Runtime apt-зависимости в Stage 2 (libpq5 для postgres, etc.)

## Чек-лист перед production

- [ ] Запинить базовый образ digest-ом (`python:3.13.1@sha256:...`)
- [ ] Добавить `.dockerignore` (есть рядом)
- [ ] Проверить размер: `docker images`
- [ ] Просканировать: `trivy image myapp:dev`
- [ ] Линтер: `hadolint Dockerfile`

## См. также

- [Этап 02 — Dockerfile](../../stages/02-dockerfile.md)
- [prompts/01-dockerfile-review.md](../../prompts/01-dockerfile-review.md)
- [prompts/02-image-slim.md](../../prompts/02-image-slim.md)
