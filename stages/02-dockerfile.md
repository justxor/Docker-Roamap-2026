# 📝 Этап 02. Dockerfile — production-уровень

> **Цель:** уметь писать Dockerfile, который собирается быстро, образ маленький, безопасный и воспроизводимый.

---

## ⏱️ Длительность

5-7 дней

## 📋 Чеклист готовности

- [ ] Знаю все инструкции Dockerfile и когда какую использовать
- [ ] Умею использовать multi-stage builds
- [ ] Понимаю BuildKit и кэш слоёв
- [ ] Умею делать multi-arch образы (buildx)
- [ ] Знаю как уменьшить размер образа в 5-10 раз
- [ ] Использую non-root user, healthcheck, ENTRYPOINT правильно
- [ ] Использую cache mounts и secrets

---

## 1. Базовый Dockerfile

```dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.13-slim-bookworm

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

Уже неплохо, но улучшим.

## 2. Multi-stage — главный паттерн

Разделяем сборку и runtime.

```dockerfile
# syntax=docker/dockerfile:1.7

# === Build stage ===
FROM python:3.13-slim-bookworm AS builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    pip install uv && uv sync --frozen --no-dev

# === Runtime stage ===
FROM python:3.13-slim-bookworm AS runtime

# Non-root user
RUN groupadd -r app && useradd -r -g app -u 10001 app

# Только нужные runtime-зависимости
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
 && rm -rf /var/lib/apt/lists/* \
 && apt-get clean

WORKDIR /app
COPY --from=builder --chown=app:app /app/.venv /app/.venv
COPY --chown=app:app . .

ENV PATH="/app/.venv/bin:\${PATH}" \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

USER app
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -fsS http://localhost:8000/health || exit 1

ENTRYPOINT ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

## 3. BuildKit — современный билдер

BuildKit включён по умолчанию в Docker 23+. Даёт:
- Параллельную сборку независимых stages
- Cache mounts: `--mount=type=cache,target=/path`
- Secrets: `--mount=type=secret,id=npmrc`
- SSH forwarding: `--mount=type=ssh`
- Build attestations (provenance, SBOM)
- Lockfile-aware кэш

```bash
# Включить вручную
DOCKER_BUILDKIT=1 docker build .

# Просмотр прогресса
docker build --progress=plain .
```

## 4. Кэш слоёв — главная экономия времени

**Правило:** часто меняющиеся файлы — в конце Dockerfile.

```dockerfile
# ❌ ПЛОХО: при изменении кода — переустановка зависимостей
COPY . .
RUN pip install -r requirements.txt

# ✅ ХОРОШО
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

### Cache mount для package manager

```dockerfile
# pip
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# apt (требует rm /var/lib/apt/lists/* + sharing=locked)
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y curl

# npm
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# uv (рекомендованный для Python в 2026)
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen
```

## 5. Secrets в build-time

Никогда не клади секреты слоями! Они навсегда останутся в образе.

```dockerfile
# ❌ Секрет остаётся в слое навсегда
ARG GITHUB_TOKEN
RUN git clone https://\${GITHUB_TOKEN}@github.com/...

# ✅ BuildKit secrets — НЕ попадают в слои
RUN --mount=type=secret,id=gh_token \
    git clone https://\$(cat /run/secrets/gh_token)@github.com/...
```

Запуск:
```bash
docker build --secret id=gh_token,env=GITHUB_TOKEN .
```

## 6. Multi-arch — buildx

В 2026 нужны минимум linux/amd64 и linux/arm64 (Apple Silicon, AWS Graviton).

```bash
# Создать билдер
docker buildx create --name multi --use --bootstrap

# Билд + push в registry за один раз
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/user/app:v1.0.0 \
  --push .
```

## 7. Distroless / Chainguard / Wolfi — минимальные базы

```dockerfile
# Distroless (Google) — нет shell
FROM gcr.io/distroless/python3-debian12:nonroot

# Chainguard — современнее, минимальный CVE-surface
FROM cgr.dev/chainguard/python:latest

# Wolfi — undistro от Chainguard
FROM cgr.dev/chainguard/wolfi-base
```

**Результат:** 30-50 MB вместо 300-500 MB, минимум CVE, нет shell для exploits.

**Цена:** нельзя `docker exec sh` для дебага → нужны другие инструменты (kubectl debug, ephemeral containers).

## 8. Pinning — воспроизводимость

```dockerfile
# ❌ не воспроизводимо
FROM python:3.13

# ✅ tag-pinned
FROM python:3.13.1-slim-bookworm

# ✅✅ digest-pinned — самое надёжное
FROM python:3.13.1-slim-bookworm@sha256:abc123def456...

# apt с версиями
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl=7.88.1-10+deb12u5 \
 && rm -rf /var/lib/apt/lists/*
```

## 9. .dockerignore

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
README.md
docs/
tests/
.github/
```

Уменьшает build context (быстрее) и предотвращает утечки секретов.

## 10. Чеклист production-Dockerfile

- [ ] `# syntax=docker/dockerfile:1.7` на первой строке
- [ ] Многоступенчатый билд (builder → runtime)
- [ ] Base image pinned by tag (или digest)
- [ ] `--no-install-recommends` + cleanup для apt
- [ ] Cache mounts для package managers
- [ ] Non-root user (`USER` с UID > 10000)
- [ ] `HEALTHCHECK` или k8s-probes
- [ ] `ENTRYPOINT` (не CMD для main process)
- [ ] `.dockerignore` есть
- [ ] Нет секретов в ARG / ENV
- [ ] Минимальный базовый образ (slim / distroless / chainguard)
- [ ] LABEL с метаданными (org.opencontainers.image.*)
- [ ] EXPOSE документирует порты
- [ ] Размер итогового образа измерен (`docker images`)

## 11. OCI labels

```dockerfile
LABEL org.opencontainers.image.source="https://github.com/user/repo"
LABEL org.opencontainers.image.description="My API"
LABEL org.opencontainers.image.licenses="MIT"
LABEL org.opencontainers.image.version="1.2.3"
LABEL org.opencontainers.image.revision="\${GIT_SHA}"
```

GitHub использует `org.opencontainers.image.source` чтобы связать пакет с репо.

---

## 🧪 Практика

### Упражнение 1: уменьши образ
Возьми любой Python/Node проект, напиши Dockerfile. Запусти `docker images` — запиши размер. Теперь оптимизируй (multi-stage + slim) — должно стать в 5+ раз меньше.

### Упражнение 2: dive
Установи [dive](https://github.com/wagoodman/dive). Запусти `dive my-image:tag` — посмотри что в каждом слое, найди мусор.

### Упражнение 3: hadolint
Установи [hadolint](https://github.com/hadolint/hadolint). Проверь свой Dockerfile, исправь все warnings.

### Упражнение 4: cache mounts
Перепиши Dockerfile с cache mounts. Сравни время повторного билда (с `docker build --no-cache` и без).

### Упражнение 5: multi-arch
Сделай buildx-билд на linux/amd64 и linux/arm64, запушь в GHCR.

### Упражнение 6: distroless
Конвертируй runtime stage в distroless. Заставь работать.

---

## 📚 Ресурсы

### Telegram
- 🔥 [ai_machinelearning_big_data](https://t.me/ai_machinelearning_big_data)
- 🐍 [pythonl](https://t.me/pythonl)
- 📚 [Папка каналов](https://t.me/addlist/8vDUwYRGujRmZjFi)

### Документация
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [BuildKit](https://docs.docker.com/build/buildkit/)
- [Best practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

### Tools
- [hadolint](https://github.com/hadolint/hadolint) — Dockerfile linter
- [dive](https://github.com/wagoodman/dive) — explore image layers
- [docker-slim](https://github.com/slimtoolkit/slim) — auto slim
- [trivy](https://github.com/aquasecurity/trivy) — vulnerability scanner

### Базовые образы
- [chainguard images](https://images.chainguard.dev/)
- [distroless](https://github.com/GoogleContainerTools/distroless)

---

## ➡️ Что дальше

[Этап 03 — Compose](03-compose.md): собираем мульти-сервисные стеки.

[⬅ MAP](../MAP.md) · [⬅ README](../README.md)
