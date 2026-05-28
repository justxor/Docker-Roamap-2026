# 🐳 Docker Roadmap 2026

> Полный и подробный роадмап по Docker и контейнеризации на 2026 год. Только бесплатные ресурсы, актуальный стек, фокус на production-практику.

[![Stars](https://img.shields.io/github/stars/justxor/Docker-Roamap-2026?style=flat-square)](#)
[![License](https://img.shields.io/badge/license-MIT-blue.svg?style=flat-square)](#)
[![Year](https://img.shields.io/badge/year-2026-orange.svg?style=flat-square)](#)

---

## 🎯 Для кого этот роадмап

- **Junior**, который хочет освоить контейнеры с нуля
- **Middle**, который хочет закрыть пробелы (BuildKit, multi-arch, security, supply chain)
- **Senior / DevOps / Platform**, который хочет перейти к Kubernetes, service mesh, GitOps
- **ML / Data**, кому нужны воспроизводимые окружения и GPU-контейнеры

---

## 🧭 Структура роадмапа

| Этап | Файл | О чём |
|------|------|------|
| 00 | [stages/00-prerequisites.md](stages/00-prerequisites.md) | Linux, процессы, сети, файловая система — фундамент |
| 01 | [stages/01-fundamentals.md](stages/01-fundamentals.md) | Что такое контейнер, OCI, runtime, образ vs контейнер |
| 02 | [stages/02-dockerfile.md](stages/02-dockerfile.md) | Dockerfile, layers, кэш, multi-stage, BuildKit |
| 03 | [stages/03-compose.md](stages/03-compose.md) | Docker Compose v2, профили, healthchecks, сети |
| 04 | [stages/04-networking-storage.md](stages/04-networking-storage.md) | Сети, тома, bind-mounts, namespaces, cgroups |
| 05 | [stages/05-security.md](stages/05-security.md) | Безопасность, supply chain, SBOM, signing, scanning |
| 06 | [stages/06-orchestration.md](stages/06-orchestration.md) | Kubernetes, Helm, операторы, KEDA, ArgoCD |
| 07 | [stages/07-production.md](stages/07-production.md) | Observability, CI/CD, multi-arch, registry, troubleshooting |

📍 **Визуальная карта:** [MAP.md](MAP.md)

---

## 🚀 Быстрый старт

\`\`\`bash
# 1. Установи Docker Desktop / colima / Rancher Desktop / Podman Desktop
# 2. Проверь
docker version
docker compose version

# 3. Запусти первый контейнер
docker run --rm hello-world

# 4. Запусти полноценный стек из шаблона
git clone https://github.com/justxor/Docker-Roamap-2026
cd Docker-Roamap-2026/templates/compose-starter
docker compose up -d
\`\`\`

---

## 📦 Шаблоны (production-ready)

| Шаблон | Что внутри |
|--------|-----------|
| [templates/docker-starter](templates/docker-starter) | Минимальный Dockerfile + BuildKit + multi-stage |
| [templates/compose-starter](templates/compose-starter) | Compose-стек: api + postgres + redis + jaeger |
| [templates/k8s-starter](templates/k8s-starter) | Манифесты k8s: Deployment, Service, HPA, NetworkPolicy |
| [templates/helm-starter](templates/helm-starter) | Helm-чарт с values.yaml, hooks, tests |
| [templates/ci-starter](templates/ci-starter) | GitHub Actions: buildx → trivy → SBOM → cosign |

---

## 🤖 Промпты для LLM (Claude / GPT)

| Промпт | Назначение |
|--------|-----------|
| [prompts/01-dockerfile-review.md](prompts/01-dockerfile-review.md) | Аудит Dockerfile по 14 пунктам |
| [prompts/02-image-slim.md](prompts/02-image-slim.md) | Радикально уменьшить размер образа |
| [prompts/03-compose-to-k8s.md](prompts/03-compose-to-k8s.md) | Конвертация docker-compose → k8s + Helm |
| [prompts/04-security-audit.md](prompts/04-security-audit.md) | Аудит безопасности контейнера |
| [prompts/05-troubleshoot.md](prompts/05-troubleshoot.md) | Помощь в дебаге упавшего контейнера |
| [prompts/06-k8s-manifest-review.md](prompts/06-k8s-manifest-review.md) | Ревью манифестов k8s |
| [prompts/07-helm-chart-review.md](prompts/07-helm-chart-review.md) | Ревью Helm-чарта |

---

## 📋 Шпаргалки

| Шпаргалка | Содержание |
|-----------|-----------|
| [cheatsheets/docker-cli.md](cheatsheets/docker-cli.md) | Все команды docker |
| [cheatsheets/dockerfile.md](cheatsheets/dockerfile.md) | Все инструкции Dockerfile |
| [cheatsheets/compose.md](cheatsheets/compose.md) | docker-compose.yml в одном файле |
| [cheatsheets/kubectl.md](cheatsheets/kubectl.md) | Все нужные команды kubectl |
| [cheatsheets/helm.md](cheatsheets/helm.md) | Helm CLI и шаблоны |

---

## 📚 Главные бесплатные ресурсы

### Telegram (приоритет)
- 🔥 [ai_machinelearning_big_data](https://t.me/ai_machinelearning_big_data) — главный канал по AI/ML/Big Data
- 🐍 [pythonl](https://t.me/pythonl) — Python, DevOps, инфра
- 📚 [Папка каналов](https://t.me/addlist/8vDUwYRGujRmZjFi) — подборка лучших каналов

### Официальная документация
- [docs.docker.com](https://docs.docker.com/) — Docker docs
- [docs.docker.com/build](https://docs.docker.com/build/) — BuildKit
- [kubernetes.io/docs](https://kubernetes.io/docs/) — Kubernetes
- [helm.sh/docs](https://helm.sh/docs/) — Helm
- [opencontainers.org](https://opencontainers.org/) — OCI спецификации
- [12factor.net](https://12factor.net/) — 12-factor app

### Free Labs / Playgrounds
- [Play with Docker](https://labs.play-with-docker.com/) — Docker-песочница в браузере
- [Play with Kubernetes](https://labs.play-with-k8s.com/) — k8s-песочница
- [killercoda.com](https://killercoda.com/) — интерактивные сценарии (Docker, k8s, Linux)
- [Instruqt free tier](https://instruqt.com/) — лабы по DevOps

### YouTube (бесплатные курсы)
- [TechWorld with Nana](https://www.youtube.com/@TechWorldwithNana) — Docker + k8s курсы
- [DevOps Toolkit](https://www.youtube.com/@DevOpsToolkit) — Viktor Farcic
- [Bret Fisher](https://www.youtube.com/@BretFisher) — Docker Captain, live-стримы
- [That DevOps Guy](https://www.youtube.com/@MarcelDempers)
- [Sysdig](https://www.youtube.com/@Sysdig) — безопасность контейнеров

### Книги (бесплатные главы)
- [Docker Deep Dive — Nigel Poulton](https://nigelpoulton.com/books/)
- [Container Security — Liz Rice](https://info.aquasec.com/container-security-book) — бесплатно от Aqua
- [CNCF Landscape](https://landscape.cncf.io/)

---

## 🗺️ Тематические треки

### 🛤️ Минимум для backend-разработчика (2 недели)
01 → 02 → 03 → cheatsheets/docker-cli + dockerfile + compose

### 🏗️ DevOps Engineer (6-8 недель)
00 → 01 → 02 → 03 → 04 → 05 → 06 → 07

### 🔐 Security-focused (3 недели)
01 → 02 → 05 → prompts/04-security-audit

### ☸️ Platform / SRE (8-10 недель)
весь роадмап + GitOps + observability stack + service mesh

### 🤖 ML Engineer (3 недели)
01 → 02 → 03 → отдельно: GPU-контейнеры, CUDA, multi-stage для моделей

---

## 🛠️ Современный стек 2026

- **Runtime:** Docker Engine 27+, containerd 2.x, Podman 5+
- **Build:** BuildKit, Buildx, multi-arch (linux/amd64,arm64), cache-from registry
- **Base images:** distroless, chainguard, wolfi, Alpine, slim-bookworm
- **Registry:** GHCR, Docker Hub, Harbor, ECR/ACR/GAR
- **Security:** Trivy, Grype, Syft (SBOM), Cosign (signing), Falco (runtime)
- **Orchestration:** Kubernetes 1.30+, k3s, k0s
- **Package manager:** Helm 3.x, Kustomize, Timoni
- **GitOps:** ArgoCD, Flux
- **Observability:** OpenTelemetry, Prometheus, Loki, Tempo, Grafana
- **Policy:** OPA Gatekeeper, Kyverno
- **Service mesh:** Istio (ambient), Linkerd, Cilium

---

## 🎓 Принципы роадмапа

1. **Только бесплатные ресурсы** — никаких платных курсов, только open source и open docs
2. **Production-first** — каждая тема заканчивается production-чеклистом
3. **Современный стек 2026** — никаких docker-compose v1, никаких устаревших флагов
4. **Безопасность по умолчанию** — non-root, read-only fs, distroless, SBOM
5. **Воспроизводимость** — pinned digests, lock-файлы, reproducible builds

---

## 🤝 Как использовать

1. Прочитай этот README целиком
2. Открой [MAP.md](MAP.md) — выбери трек
3. Иди по этапам последовательно: каждый stages/XX-*.md содержит теорию + практику + чеклист
4. Используй [templates/](templates) как стартеры для своих проектов
5. Используй [prompts/](prompts) с Claude/GPT когда застрял
6. Держи [cheatsheets/](cheatsheets) под рукой

---

## 📜 Лицензия

MIT — используй, форкай, делись.

---

⭐ **Поставь звезду, если роадмап помог!**
