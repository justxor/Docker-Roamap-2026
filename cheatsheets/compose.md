# 🎼 Docker Compose Cheatsheet

`docker-compose.yml` в одном файле.

---

## Минимальный stack

```yaml
name: myapp

services:
  api:
    build: .
    image: ghcr.io/user/api:dev
    ports: ["8000:8000"]
    environment:
      DATABASE_URL: postgresql://app:app@db:5432/app
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:8000/health"]
      interval: 10s
      timeout: 3s
      retries: 3

  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: app
    volumes: [db-data:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s

volumes:
  db-data:
```

---

## Ключевые фичи

### Profiles
```yaml
services:
  jaeger:
    image: jaegertracing/all-in-one
    profiles: ["observability"]
```
```bash
docker compose --profile observability up -d
```

### Watch (hot reload)
```yaml
services:
  api:
    develop:
      watch:
        - { action: sync, path: ./src, target: /app/src }
        - { action: rebuild, path: ./pyproject.toml }
```
```bash
docker compose watch
```

### Secrets
```yaml
services:
  api:
    secrets: [db_password]
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

### Networks
```yaml
services:
  api:
    networks: [frontend, backend]
  db:
    networks: [backend]

networks:
  frontend:
  backend:
    internal: true
```

### Resource limits
```yaml
services:
  api:
    deploy:
      resources:
        limits:
          cpus: "1.5"
          memory: 512M
```

---

## CLI

```bash
docker compose up -d                       # запустить
docker compose up -d --build               # + ребилд
docker compose --profile observability up   # с профилем
docker compose down                         # остановить + удалить
docker compose down -v                      # + волюмы
docker compose ps                           # статус
docker compose logs -f --tail=100 api
docker compose exec api bash                # войти
docker compose run --rm api alembic upgrade head
docker compose watch                        # hot reload
docker compose config                       # rendered config
docker compose pull                         # обновить images
docker compose restart api
```

---

## Override-файлы

```bash
# Авто: docker-compose.override.yml подхватывается
docker compose up

# Явно
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

---

[⬅ README](../README.md)
