# 📦 Этап 01. Fundamentals — что такое контейнер

> **Цель:** понять что такое контейнер, OCI-стандарт, разницу image vs container, runtime-стек (containerd, runc).

---

## ⏱️ Длительность

2-3 дня

## 📋 Чеклист готовности

- [ ] Могу объяснить разницу VM vs контейнер
- [ ] Понимаю что такое OCI image spec и OCI runtime spec
- [ ] Знаю слои стека: Docker CLI → dockerd → containerd → runc
- [ ] Понимаю разницу image / container / layer / tag / digest
- [ ] Умею работать с registry (push/pull, теги, дайджесты)
- [ ] Знаю альтернативы Docker: Podman, containerd, nerdctl

---

## 1. Контейнер vs VM

| | VM | Container |
|--|----|----|
| Ядро | своё | общее с хостом |
| Загрузка | минуты | миллисекунды |
| Размер | гигабайты | мегабайты |
| Изоляция | сильная (гипервизор) | слабее (namespaces) |
| Density | десятки на хост | тысячи на хост |

**Главное:** контейнер не виртуализирует железо. Это обычный Linux-процесс в namespaces + cgroups + chroot-подобной FS.

## 2. OCI — Open Container Initiative

Это стандарты, которые делают экосистему единой.

- **OCI Image Spec** — формат образа (манифест, конфиг, слои)
- **OCI Runtime Spec** — как запускать (config.json, bundle)
- **OCI Distribution Spec** — как registry отдаёт образы

Поэтому образ Docker можно запустить через Podman, containerd, CRI-O — все они OCI-совместимы.

## 3. Стек Docker

```
docker (CLI)
   │  REST API
   ▼
dockerd (daemon)
   │  gRPC
   ▼
containerd (high-level runtime)
   │
   ▼
runc / crun / youki (low-level runtime, OCI)
   │
   ▼
ядро Linux (namespaces + cgroups)
```

В 2026 Kubernetes использует containerd напрямую (Docker shim давно удалён).

## 4. Image vs Container

- **Image** — read-only снэпшот, набор слоёв + конфиг. Идентифицируется через **digest** (sha256:...).
- **Container** — запущенный (или остановленный) экземпляр образа + writable layer сверху.
- **Layer** — diff файловой системы между шагами Dockerfile.
- **Tag** — человекочитаемая метка (latest, v1.2.3). Изменяема!
- **Digest** — неизменяемый хэш контента. Используй в production.

```bash
# Тег vs дайджест
docker pull nginx:1.27                                             # тег
docker pull nginx@sha256:abc123...                                 # дайджест (production!)

# Посмотреть слои
docker history nginx:1.27
docker inspect nginx:1.27
```

## 5. Registry

Где хранятся образы:
- **Docker Hub** (hub.docker.com) — по умолчанию, есть rate limits для анонимов
- **GHCR** (ghcr.io) — GitHub Container Registry, бесплатно для public
- **Harbor** — self-hosted enterprise registry
- **Quay** (quay.io) — Red Hat
- **ECR / GCR / ACR** — облака

```bash
# Полное имя образа
docker pull ghcr.io/user/app:v1.0.0
# = registry / repository / tag

# Логин
docker login ghcr.io
# (для GHCR — Personal Access Token со scope read:packages)

# Push
docker tag myapp:dev ghcr.io/user/myapp:v1.0.0
docker push ghcr.io/user/myapp:v1.0.0
```

## 6. Альтернативы Docker

- **Podman** — daemonless, rootless, drop-in replacement для docker CLI
- **containerd + nerdctl** — то, что использует k8s, можно работать напрямую
- **Buildah** — только сборка образов
- **Skopeo** — копирование/инспекция образов без docker daemon
- **Kaniko** — сборка в самом k8s без docker daemon
- **BuildKit** — современный builder (используется Docker и standalone)

```bash
# Podman практически совместим
podman run --rm hello-world
alias docker=podman   # если хочется
```

## 7. Где живут образы локально

```bash
# Docker storage driver
docker info | grep -i 'Storage Driver'
# overlay2 — современный, по умолчанию

# Где лежит
sudo ls /var/lib/docker/   # на Linux
# overlay2/, image/, containers/, volumes/...
```

---

## 🧪 Практика

### Упражнение 1: первый контейнер
```bash
docker run --rm -it alpine sh
# внутри:
ps aux             # увидишь только свои процессы (PID namespace)
cat /etc/os-release
ip a               # своя сеть
```

### Упражнение 2: слои
```bash
docker pull nginx:1.27
docker history nginx:1.27 --no-trunc
docker inspect nginx:1.27 | jq '.[0].RootFS.Layers'
```

### Упражнение 3: digest pinning
Найди digest своего образа, запиши в `docker pull image@sha256:...` и убедись что pull детерминирован.

### Упражнение 4: GHCR
Зарегистрируй PAT, залогинься, запушь свой первый образ в `ghcr.io/<твой-юзер>/hello:v1`.

### Упражнение 5: Podman
Установи Podman, запусти тот же образ. Сравни поведение.

---

## 📚 Ресурсы

### Telegram
- 🔥 [ai_machinelearning_big_data](https://t.me/ai_machinelearning_big_data)
- 🐍 [pythonl](https://t.me/pythonl)
- 📚 [Папка каналов](https://t.me/addlist/8vDUwYRGujRmZjFi)

### Документация
- [docs.docker.com/get-started](https://docs.docker.com/get-started/)
- [opencontainers.org](https://opencontainers.org/) — спецификации
- [containerd.io](https://containerd.io/)
- [podman.io](https://podman.io/)

### Видео
- [Liz Rice — What is a container?](https://www.youtube.com/watch?v=sK5i-N34im8)
- [TechWorld with Nana — Docker Crash Course](https://www.youtube.com/watch?v=pg19Z8LL06w)

---

## ➡️ Что дальше

[Этап 02 — Dockerfile](02-dockerfile.md): пишем production-Dockerfile.

[⬅ MAP](../MAP.md) · [⬅ README](../README.md)
