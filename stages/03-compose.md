# 🎼 Этап 03. Docker Compose — мульти-сервисные стеки

> **Цель:** уметь собирать полноценный локальный стек: API + БД + кэш + observability одной командой.

---

## ⏱️ Длительность

4-5 дней

## 📋 Чеклист готовности

- [ ] Знаю Compose Specification v2 (без устаревшего `version:`)
- [ ] Умею описывать depends_on с healthchecks
- [ ] Использую profiles для опциональных сервисов
- [ ] Использую `env_file`, `secrets`, `configs`
- [ ] Понимаю сети и custom networks
- [ ] Использую `watch` для dev hot-reload
- [ ] Умею `docker compose run` и `exec`

---

## 1. Compose Specification v2

В 2026 не используем `docker-compose` (Python) и не пишем `version: "3.8"`. Только `docker compose` (Go, plugin) и top-level keys.

```yaml
# docker-compose.yml
name: myproject

services:
  api:
    build:
      context: .
      target: runtime
    image: ghcr.io/user/myapp:dev
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql+asyncpg://user:pass@db:5432/app
      REDIS_URL: redis://redis:6379/0
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:8000/health"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 10s

  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: app
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d app"]
      interval: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --save 60 1 --loglevel warning
    volumes:
      - redis-data:/data

volumes:
  db-data:
  redis-data:
```

```bash
docker compose up -d
docker compose logs -f api
docker compose ps
docker compose down       # -v добавит удаление volumes
```

## 2. Healthchecks + depends_on

**Главное:** `depends_on` без `condition: service_healthy` ждёт только запуска контейнера, а не готовности приложения.

```yaml
depends_on:
  db:
    condition: service_healthy        # ждать пока healthcheck = healthy
    restart: true                     # перезапустить api если db рестартует
```

Виды condition:
- `service_started` — контейнер запущен
- `service_healthy` — healthcheck прошёл
- `service_completed_successfully` — exit 0 (для one-shot задач)

## 3. Profiles — опциональные сервисы

```yaml
services:
  api: { ... }
  db: { ... }

  jaeger:
    image: jaegertracing/all-in-one:latest
    profiles: ["observability"]
    ports: ["16686:16686"]

  pgadmin:
    image: dpage/pgadmin4
    profiles: ["tools"]
```

```bash
docker compose up -d                              # api + db
docker compose --profile observability up -d      # + jaeger
docker compose --profile observability --profile tools up -d
```

## 4. Сети

По умолчанию Compose создаёт сеть `<project>_default`. Сервисы видят друг друга по имени.

```yaml
services:
  api:
    networks: [frontend, backend]
  db:
    networks: [backend]
  nginx:
    networks: [frontend]

networks:
  frontend:
  backend:
    internal: true        # без выхода в интернет
```

## 5. Volumes vs bind mounts

```yaml
services:
  api:
    volumes:
      - ./app:/app                    # bind mount: dev hot-reload
      - /app/.venv                    # anon volume: исключить из bind
      - cache:/app/.cache             # named volume
      - type: tmpfs
        target: /tmp
        tmpfs: { size: 100M }

volumes:
  cache:
```

## 6. Secrets и Configs

```yaml
services:
  api:
    secrets:
      - db_password
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

Не путать с переменными окружения! Secret монтируется как файл — это безопаснее (нет в `env`, нет в `ps`).

## 7. Watch — hot reload в Compose v2

```yaml
services:
  api:
    develop:
      watch:
        - action: sync
          path: ./src
          target: /app/src
        - action: rebuild
          path: ./pyproject.toml
```

```bash
docker compose watch
# меняешь src/ → автоматически синкается в контейнер
# меняешь pyproject.toml → ребилд
```

## 8. Override-файлы

```bash
# По умолчанию подхватывается docker-compose.override.yml
docker compose up

# Или явно
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

Структура:
- `docker-compose.yml` — общие настройки
- `docker-compose.override.yml` — для dev (volumes, debug ports)
- `docker-compose.prod.yml` — для prod (без debug, разные ENV)

## 9. Run, exec, logs

```bash
# Одноразовая задача
docker compose run --rm api alembic upgrade head

# Войти в контейнер
docker compose exec api bash

# Логи
docker compose logs -f --tail=100 api

# Перезапуск с обновлением
docker compose up -d --build api

# Очистка
docker compose down -v --remove-orphans
```

## 10. Production?

Compose — для **локального dev и CI**. В production используй Swarm (умирает) или Kubernetes.

Но Compose Stack может работать в Swarm через `docker stack deploy` — для маленьких проектов это норм.

---

## 🧪 Практика

### Упражнение 1: API + DB
Напиши compose-файл: FastAPI/Express + PostgreSQL + Redis. С healthchecks и правильными depends_on.

### Упражнение 2: hot reload
Добавь `develop.watch` для своего проекта. Проверь что изменения подхватываются без рестарта.

### Упражнение 3: profiles
Сделай профили: `dev` (debug-инструменты), `observability` (jaeger), `tools` (pgadmin, redis-commander).

### Упражнение 4: secrets
Перенеси пароли БД в `secrets:` (через файл). Удостоверься, что в `docker compose config` они не светятся.

### Упражнение 5: override
Сделай `docker-compose.override.yml` для dev (порты, volumes) и `docker-compose.prod.yml` для prod.

---

## 📚 Ресурсы

### Telegram
- 🔥 [ai_machinelearning_big_data](https://t.me/ai_machinelearning_big_data)
- 🐍 [pythonl](https://t.me/pythonl)
- 📚 [Папка каналов](https://t.me/addlist/8vDUwYRGujRmZjFi)

### Документация
- [Compose Specification](https://compose-spec.io/)
- [docs.docker.com/compose](https://docs.docker.com/compose/)
- [Compose watch](https://docs.docker.com/compose/file-watch/)

### Tools
- [awesome-compose](https://github.com/docker/awesome-compose) — готовые шаблоны
- [dozzle](https://dozzle.dev/) — UI для логов
- [lazydocker](https://github.com/jesseduffield/lazydocker) — TUI

---

## ➡️ Что дальше

[Этап 04 — Networking & Storage](04-networking-storage.md): глубоко в сети и тома.

[⬅ MAP](../MAP.md) · [⬅ README](../README.md)
