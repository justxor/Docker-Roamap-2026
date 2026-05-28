# 🪶 Prompt: радикально уменьшить образ

Используй когда твой образ весит 500MB и больше — LLM поможет сжать в 5-10 раз.

---

## Промпт

Ты — эксперт по оптимизации Docker-образов. Цель — уменьшить размер моего образа в 5+ раз без потери функциональности.

Проанализируй и дай план:

1. **Базовый образ** — переход на slim / alpine / distroless / chainguard / wolfi. Оцени экономию в MB.
2. **Multi-stage** — вынеси build-deps (compilers, headers) в builder stage.
3. **Layer caching** — правильный порядок, cache mounts.
4. **apt cleanup** — `--no-install-recommends`, `rm -rf /var/lib/apt/lists/*`.
5. **Ненужные файлы** — что из builder не нужно в runtime (.git, tests/, docs/, build artifacts).
6. **`.dockerignore`** — что добавить.
7. **Pip/npm caches** — `--no-cache-dir`, `npm ci --omit=dev`.
8. **Python bytecode** — или включить (UV_COMPILE_BYTECODE=1) или выключить (PYTHONDONTWRITEBYTECODE=1).
9. **Static бинарь** — для Go/Rust — scratch или distroless/static.
10. **`docker-slim`** — рекомендуй если ручная оптимизация не помогает.

Сформат ответа:
- Текущий размер (спроси у меня, если не указан)
- Топ-5 источников веса (в MB)
- Новый Dockerfile целиком
- Ожидаемый размер после
- Команды для верификации (`docker images`, `dive image`)

---

## Мой Dockerfile

```dockerfile
<вставь свой Dockerfile>
```

Текущий размер: <введи в MB>

---

## См. также

- [stages/02-dockerfile.md](../stages/02-dockerfile.md)
- [prompts/01-dockerfile-review.md](01-dockerfile-review.md)
