# 📝 Dockerfile Cheatsheet

Все инструкции Dockerfile + best practices.

---

## Инструкции

| Инструкция | Назначение |
|------------|------------|
| `FROM` | базовый образ |
| `ARG` | build-time переменная |
| `ENV` | runtime переменная |
| `WORKDIR` | cwd в контейнере |
| `COPY` | копирование с host или stage |
| `ADD` | копирование + URL/tar (редко нужно) |
| `RUN` | выполнить команду в build-time |
| `CMD` | default команда |
| `ENTRYPOINT` | main команда (для signal handling) |
| `EXPOSE` | документирует порт |
| `VOLUME` | mount point |
| `USER` | переключить пользователя |
| `HEALTHCHECK` | проверка живости |
| `LABEL` | метаданные (OCI) |
| `STOPSIGNAL` | сигнал для stop |
| `SHELL` | shell для RUN |
| `ONBUILD` | инструкция для children images |

---

## Минимальный production-Dockerfile

```dockerfile
# syntax=docker/dockerfile:1.7

ARG PYTHON_VERSION=3.13

# Builder
FROM python:${PYTHON_VERSION}-slim-bookworm AS builder
WORKDIR /app
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir -r requirements.txt
COPY . .

# Runtime
FROM python:${PYTHON_VERSION}-slim-bookworm AS runtime
RUN groupadd -r app && useradd -r -g app --uid 10001 app
WORKDIR /app
COPY --from=builder --chown=app:app /app /app
USER 10001
EXPOSE 8000
HEALTHCHECK --interval=30s CMD curl -fsS http://localhost:8000/health || exit 1
ENTRYPOINT ["python", "-m", "app"]
```

---

## BuildKit features

```dockerfile
# Cache mount (pip / npm / apt)
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Secret mount (НЕ попадает в слои)
RUN --mount=type=secret,id=gh_token \
    git clone https://$(cat /run/secrets/gh_token)@github.com/...

# SSH mount
RUN --mount=type=ssh git clone git@github.com:user/repo.git

# Bind mount build context
RUN --mount=type=bind,source=.,target=/src \
    cp -r /src/dist /app
```

---

## Multi-arch

```bash
docker buildx create --name multi --use --bootstrap
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/user/img:v1 \
  --push .
```

---

## .dockerignore

```
.git
.venv
__pycache__
*.pyc
node_modules
.env
.env.*
*.log
.idea
.vscode
tests/
docs/
.github/
```

---

## OCI labels

```dockerfile
LABEL org.opencontainers.image.source="https://github.com/user/repo"
LABEL org.opencontainers.image.description="My API"
LABEL org.opencontainers.image.licenses="MIT"
LABEL org.opencontainers.image.version="1.2.3"
LABEL org.opencontainers.image.revision="${GIT_SHA}"
```

---

## Best practices

- `# syntax=docker/dockerfile:1.7` на первой строке
- Pin base image: `python:3.13.1-slim-bookworm@sha256:...`
- Multi-stage: builder + runtime
- `--no-install-recommends` + `rm -rf /var/lib/apt/lists/*`
- Cache mounts для package managers
- Non-root user (UID > 10000)
- Read-only root FS в runtime
- ENTRYPOINT в exec form (`["cmd", "arg"]`)
- HEALTHCHECK для Docker, probes для k8s
- .dockerignore всегда

---

[⬅ README](../README.md)
