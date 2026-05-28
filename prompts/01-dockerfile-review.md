# 🔍 Prompt: ревью Dockerfile по 14 пунктам

Используй с Claude / GPT когда нужно проверить Dockerfile перед деплоем.

---

## Промпт

Ты — staff DevOps инженер с 10+ лет опыта в Docker и production-окружениях. Проведи ревью этого Dockerfile по 14 пунктам:

1. **Base image** — pinned by tag (а лучше digest)? Slim / distroless / chainguard?
2. **Layer order** — сначала зависимости (медленно меняются), потом код (быстро меняется)?
3. **Multi-stage** — есть отдельный builder и runtime?
4. **apt-get** — `--no-install-recommends` + `rm -rf /var/lib/apt/lists/*` + `apt-get clean`?
5. **Cache mounts** — `--mount=type=cache` для pip/npm/uv/apt?
6. **Secrets** — нет секретов в ARG/ENV? Используется `--mount=type=secret` для build-time?
7. **Non-root user** — `USER` с UID > 10000?
8. **WORKDIR** — указан и не `/` или `/tmp`?
9. **HEALTHCHECK** — есть и проверяет реальную готовность приложения?
10. **EXPOSE** — документирует все нужные порты?
11. **ENTRYPOINT vs CMD** — main-процесс через ENTRYPOINT (для proper signal handling)?
12. **OCI labels** — `org.opencontainers.image.source/description/version`?
13. **Reproducibility** — версии пакетов запинены? `apt-get install curl=X.Y.Z`?
14. **Размер** — оценка финального размера (вместо подсчёта дай recommendation)?

По каждому пункту дай:
- ✅ OK / ⚠️ Warning / ❌ Critical
- Конкретную строку Dockerfile с проблемой
- Исправленный фрагмент кода

В конце:
- Топ-3 самых критичных проблемы
- Полный исправленный Dockerfile
- Прогноз размера до/после оптимизации

---

## Мой Dockerfile

```dockerfile
<вставь свой Dockerfile>
```

---

## См. также

- [stages/02-dockerfile.md](../stages/02-dockerfile.md)
- [templates/docker-starter](../templates/docker-starter/) — эталонный Dockerfile
- [prompts/02-image-slim.md](02-image-slim.md) — уменьшить размер
