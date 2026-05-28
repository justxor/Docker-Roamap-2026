# 🐳 Docker CLI Cheatsheet

Все нужные команды `docker` в одном месте.

---

## Контейнеры

```bash
# Запуск
docker run --rm -it ubuntu bash             # интерактивно, удалить после
docker run -d --name web -p 8080:80 nginx   # detached, имя, порт
docker run --restart=unless-stopped ...     # автоперезапуск

# Просмотр
docker ps                # запущенные
docker ps -a             # все, включая остановленные
docker ps --filter status=exited
docker stats             # CPU/RAM в реальном времени

# Управление
docker stop <id>         # SIGTERM, ждёт 10s, потом SIGKILL
docker stop -t 30 <id>   # увеличить grace period
docker restart <id>
docker rm <id>           # удалить остановленный
docker rm -f <id>        # принудительно (stop + rm)

# Внутрь
docker exec -it <id> bash
docker exec <id> ls /app
docker logs -f --tail=100 <id>
docker logs --since=1h <id>
docker top <id>          # процессы внутри
docker inspect <id>      # вся метаинформация
docker port <id>         # маппинги портов
```

---

## Образы

```bash
# Сборка
docker build -t myapp:dev .
docker build --no-cache .
docker build --pull .                    # обновить базовый образ
docker build -f Dockerfile.prod .
docker build --target builder .          # остановиться на конкретном stage

# Buildx (multi-arch)
docker buildx create --name multi --use --bootstrap
docker buildx build --platform linux/amd64,linux/arm64 -t img:v1 --push .

# Просмотр
docker images
docker image ls --filter dangling=true   # untagged "<none>"
docker history nginx                     # слои образа
docker inspect nginx

# Управление
docker pull nginx:1.27
docker pull nginx@sha256:abc123...       # digest pin
docker tag myapp:dev ghcr.io/user/myapp:v1
docker push ghcr.io/user/myapp:v1
docker rmi <id>                          # удалить
docker image prune                       # удалить dangling
docker image prune -a                    # удалить ВСЕ неиспользуемые
```

---

## Volumes

```bash
docker volume create mydata
docker volume ls
docker volume inspect mydata
docker volume rm mydata
docker volume prune                      # удалить неиспользуемые

# Bind mount
docker run -v $(pwd):/app nginx
docker run -v $(pwd):/app:ro nginx       # read-only

# Named volume
docker run -v mydata:/var/lib/postgresql/data postgres

# tmpfs
docker run --tmpfs /tmp:size=100m nginx
```

---

## Networks

```bash
docker network ls
docker network create mynet
docker network create --driver bridge --subnet=10.10.0.0/16 mynet
docker network inspect mynet
docker network connect mynet <container>
docker network disconnect mynet <container>
docker network prune

# Запуск в сети
docker run --network=mynet --name=web nginx
docker run --network=host nginx          # без изоляции (только Linux)
docker run --network=none nginx          # без сети
```

---

## System

```bash
docker version
docker info
docker system df                         # сколько места занимают слои/volumes/контейнеры
docker system prune                      # удалить остановленные + dangling
docker system prune -a --volumes         # удалить ВСЁ неиспользуемое (опасно!)
docker events                            # стрим событий Docker
```

---

## Compose

```bash
docker compose up -d
docker compose up -d --build
docker compose up --profile observability
docker compose down                      # остановить + удалить контейнеры
docker compose down -v                   # + удалить volumes
docker compose ps
docker compose logs -f api
docker compose exec api bash
docker compose run --rm api alembic upgrade head
docker compose watch                     # hot reload
docker compose config                    # rendered config
docker compose pull                      # обновить образы
```

---

## Security

```bash
# Без root
docker run --user 10001:10001 nginx

# Read-only FS
docker run --read-only --tmpfs /tmp nginx

# Drop capabilities
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx

# No new privileges
docker run --security-opt no-new-privileges nginx

# Seccomp / AppArmor
docker run --security-opt seccomp=profile.json nginx

# Scan
docker scout cves nginx:latest           # встроенный сканер Docker
trivy image nginx:latest                 # альтернатива
```

---

## Полезное

```bash
# Очистка диска
docker system prune -a --volumes

# Войти в контейнер за процессом
sudo nsenter -t $(docker inspect -f '{{.State.Pid}}' <ct>) -m -n -p -u

# Копирование файлов
docker cp file.txt <ct>:/app/
docker cp <ct>:/app/log.txt ./

# Resource limits
docker run --memory=512m --cpus=1.5 nginx
docker run --oom-kill-disable nginx       # запретить OOM kill (опасно)

# Restart policy
docker run --restart=on-failure:5 nginx   # 5 попыток
docker run --restart=unless-stopped nginx

# Healthcheck snapshot
docker inspect --format='{{json .State.Health}}' <ct>
```

---

[⬅ README](../README.md)
