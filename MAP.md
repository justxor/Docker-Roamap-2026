# 🗺️ Карта Docker Roadmap 2026

```
                    ┌───────────────────────────────┐
                    │   00. PREREQUISITES (Linux)   │
                    │  процессы, сети, FS, права    │
                    └──────────────┬────────────────┘
                                   │
                    ┌──────────────▼────────────────┐
                    │   01. FUNDAMENTALS            │
                    │  OCI, runtime, image vs ctr   │
                    └──────────────┬────────────────┘
                                   │
              ┌────────────────────┼─────────────────────┐
              │                    │                     │
   ┌──────────▼─────────┐ ┌────────▼──────────┐ ┌────────▼──────────┐
   │ 02. DOCKERFILE     │ │ 03. COMPOSE       │ │ 04. NET / STORAGE │
   │ layers, BuildKit   │ │ multi-service     │ │ namespaces,cgroups│
   │ multi-stage, cache │ │ healthchecks      │ │ volumes, networks │
   └──────────┬─────────┘ └────────┬──────────┘ └────────┬──────────┘
              │                    │                     │
              └────────────────────┼─────────────────────┘
                                   │
                    ┌──────────────▼────────────────┐
                    │   05. SECURITY                │
                    │  SBOM, signing, scan, supply  │
                    └──────────────┬────────────────┘
                                   │
                    ┌──────────────▼────────────────┐
                    │   06. ORCHESTRATION (k8s)     │
                    │  Deploy, Helm, ArgoCD, KEDA   │
                    └──────────────┬────────────────┘
                                   │
                    ┌──────────────▼────────────────┐
                    │   07. PRODUCTION              │
                    │  CI/CD, observability, multi  │
                    │  arch, registry, troubleshoot │
                    └───────────────────────────────┘
```

---

## 🎯 Треки по ролям

### 👨‍💻 Backend Developer (2 недели)
```
01 ─► 02 ─► 03 ─► cheatsheets
```
**Цель:** уметь упаковать своё приложение, написать compose-стек для локалки.

### 🔧 DevOps Engineer (6-8 недель)
```
00 ─► 01 ─► 02 ─► 03 ─► 04 ─► 05 ─► 06 ─► 07
```
**Цель:** деплой в k8s, CI/CD, observability, безопасность.

### 🔐 Security Engineer (3 недели)
```
01 ─► 02 ─► 05 ─► prompts/04-security-audit
```
**Цель:** аудит образов, supply chain, runtime-защита.

### ☸️ Platform / SRE (8-10 недель)
```
весь роадмап + GitOps + service mesh + KEDA + Falco
```
**Цель:** строить платформу для команд разработки.

### 🤖 ML Engineer (3 недели)
```
01 ─► 02 (multi-stage) ─► 03 ─► GPU-контейнеры (CUDA, nvidia-container-toolkit)
```
**Цель:** воспроизводимые окружения для обучения и инференса.

---

## 📊 Сложность по этапам

| Этап | Сложность | Длительность | Pre-requisites |
|------|-----------|--------------|----------------|
| 00 | 🟢 Easy | 3-5 дней | — |
| 01 | 🟢 Easy | 2-3 дня | 00 |
| 02 | 🟡 Medium | 5-7 дней | 01 |
| 03 | 🟡 Medium | 4-5 дней | 02 |
| 04 | 🟡 Medium | 4-6 дней | 02, 03 |
| 05 | 🟠 Hard | 5-7 дней | 02, 04 |
| 06 | 🔴 Very Hard | 14-21 день | 03, 04, 05 |
| 07 | 🔴 Very Hard | 7-14 дней | весь роадмап |

---

## 🛠️ Инструменты по этапам

| Этап | Ключевые инструменты |
|------|----------------------|
| 00 | bash, ip, netstat, ss, lsof, strace, mount |
| 01 | docker, podman, containerd, runc, skopeo |
| 02 | docker buildx, BuildKit, dockerfile-linter, hadolint, dive |
| 03 | docker compose v2, traefik, dozzle |
| 04 | docker network, nsenter, iptables, cilium |
| 05 | trivy, grype, syft, cosign, falco, kyverno |
| 06 | kubectl, helm, argocd, kustomize, k9s, lens |
| 07 | github actions, prometheus, loki, tempo, grafana, opentelemetry |

---

## 🎓 Финальные проекты по треку

### Backend Developer
- Упаковать FastAPI/Django/Flask приложение
- Собрать compose-стек: app + db + redis + nginx
- Запушить в GHCR через GitHub Actions

### DevOps Engineer
- Развернуть приложение в k3s/k3d
- Написать Helm-чарт с values для dev/staging/prod
- Настроить ArgoCD на GitOps деплой
- Добавить observability (Prometheus + Grafana + Loki)

### Security Engineer
- Сделать distroless образ для legacy app
- Подписать образы Cosign
- Настроить Trivy в CI и блокировку HIGH CVE
- Запустить Falco для runtime-детекции

### Platform / SRE
- Поднять платформу с tenancy: namespaces + RBAC + NetworkPolicy
- KEDA-автоскейлинг по метрикам Kafka
- ArgoCD ApplicationSet для multi-tenant
- Sealed Secrets / External Secrets

### ML Engineer
- Multi-stage образ для PyTorch модели
- GPU-контейнер с CUDA 12.x
- KServe / Seldon для онлайн-инференса

---

## 🚦 Чеклист готовности к работе

После прохождения роадмапа ты можешь:

- [ ] Объяснить разницу image vs container, OCI vs Docker
- [ ] Написать оптимальный Dockerfile (multi-stage, кэш, размер < 100MB)
- [ ] Сделать multi-arch билды через buildx
- [ ] Собрать compose-стек с healthchecks и profiles
- [ ] Объяснить как работают cgroups и namespaces
- [ ] Сделать SBOM и подписать образ
- [ ] Найти и пофиксить CVE в образе
- [ ] Написать Helm-чарт с values для dev/staging/prod
- [ ] Настроить HPA + PDB + NetworkPolicy
- [ ] Поднять observability stack
- [ ] Дебажить упавший Pod в production

---

[⬅ Вернуться в README](README.md)
