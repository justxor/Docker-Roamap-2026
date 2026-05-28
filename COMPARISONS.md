# ⚖️ Сравнения: что выбрать в 2026

> Практические сравнительные таблицы инструментов. Без воды — только факты.

---

## 1. Container Engines

| | Docker | Podman | containerd | nerdctl |
|---|--------|--------|------------|---------|
| **Daemon** | dockerd | daemonless | containerd | containerd |
| **Rootless** | Да (настройка) | Да (дефолт) | Да | Да |
| **Compose** | docker compose | podman compose / kube play | nerdctl compose | nerdctl compose |
| **K8s pods** | Нет | Да (`podman pod`) | Нет | Нет |
| **CLI совместимость** | — | docker alias | docker-like | docker-like |
| **Buildx** | ✅ | Ограничено | — | ✅ |
| **Когда выбрать** | Default, max compat | Rootless prod, RH eco | k8s prod | Containerd CLI |

---

## 2. Base images

| Base | Размер | Shell | Пакеты | Security |
|------|---------|-------|----------|----------|
| **ubuntu:24.04** | ~78 MB | ✅ | apt | Средне |
| **debian:13-slim** | ~75 MB | ✅ | apt | Средне |
| **alpine:3.20** | ~7 MB | ash | apk | musl (внимание к glibc) |
| **distroless/static** | ~2 MB | ❌ | — | Высокая |
| **distroless/cc** | ~17 MB | ❌ | — | Высокая |
| **distroless/python3** | ~50 MB | ❌ | — | Высокая |
| **chainguard/static** | ~2 MB | ❌ | — | Максимальная, SBOM в комплекте |
| **wolfi-base** | ~16 MB | ✅ | apk | Glibc + 0-CVE ориентир |
| **scratch** | 0 | ❌ | — | Макс (ничего нет) |

**Рекомендация 2026:** `chainguard/*` или `distroless/*` для prod, `wolfi-base` когда нужен shell.

---

## 3. Реестры образов

| Registry | Self-host | Free tier | Фичи |
|----------|-----------|-----------|-------|
| **GHCR** | ❌ | Щедрый | Интеграция GitHub, OIDC |
| **Docker Hub** | ❌ | Ограничен по pull | Official Images |
| **GAR / ACR / ECR** | ❌ | Pay-as-you-go | Cloud-native |
| **Harbor** | ✅ | Free OSS | Trivy, replication, Cosign |
| **Zot** | ✅ | Free OSS | OCI-native, лёгкий |
| **Quay** | ✅ (RH) | Free public | Clair scan |
| **distribution (Docker registry)** | ✅ | Free | Минимальный |

---

## 4. K8s distributions для локальной разработки

| | minikube | kind | k3d | k3s | microk8s |
|---|----------|------|-----|-----|----------|
| **Старт** | ~60c | ~30c | ~20c | ~10c | ~40c |
| **Размер RAM** | 2 GB | 1 GB | 500 MB | 500 MB | 540 MB |
| **Multi-node** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Где работает** | VM/Docker | Docker | Docker | Linux | Linux |
| **Для чего** | Универсален | CI/CD | dev fast | edge/prod | Ubuntu IoT |

**Рекомендация:** `kind` для CI, `k3d` для локальной разработки.

---

## 5. CI/CD для контейнеров

| | GitHub Actions | GitLab CI | Jenkins | Tekton | Argo Workflows |
|---|----------------|-----------|---------|--------|----------------|
| **Hosting** | SaaS / self-host | SaaS / self-host | Self-host | k8s-native | k8s-native |
| **Free** | 2000 min/мес | 400 min/мес | Free OSS | Free OSS | Free OSS |
| **YAML** | Простой | Средний | Groovy | Сложный | Сложный |
| **k8s-native** | ❌ | ⚠️ | ⚠️ | ✅ | ✅ |
| **Рекомендация** | Default 2026 | Если GitLab | Legacy | Platform team | Complex DAG |

---

## 6. GitOps tools

| | Argo CD | Flux | Rancher Fleet |
|---|---------|------|---------------|
| **UI** | ✅ богатый | ❌ (UI отдельно) | ✅ |
| **Multi-tenancy** | ✅ (Не лучшая) | ✅ (RBAC) | ✅ (fleet) |
| **Multi-cluster** | ApplicationSets | ✅ (из коробки) | ✅ |
| **Helm/Kustomize** | ✅ | ✅ | ✅ |
| **Стандарт в 2026** | ✅ | ✅ (Flagger) | Rancher only |

---

## 7. Vulnerability scanners

| | Trivy | Grype | Clair | Snyk |
|---|-------|-------|-------|------|
| **OSS** | ✅ | ✅ | ✅ | ⚠️ (freemium) |
| **Сканирует** | image+IaC+secrets | image+SBOM | image | image+code+IaC |
| **Скорость** | ⚡⚡⚡ | ⚡⚡ | ⚡ | ⚡⚡ |
| **Интеграция** | Отличная | Anchore stack | Quay | UI dashboard |
| **Рекомендация** | ✅ Default | Anchore | Quay users | Enterprise |

---

## 8. Observability stack

| Столп | Open Source | SaaS |
|--------|-------------|------|
| **Metrics** | Prometheus + Thanos / Mimir / VictoriaMetrics | Datadog, New Relic |
| **Logs** | Loki, OpenSearch | Datadog, Splunk |
| **Traces** | Tempo, Jaeger | Honeycomb, Lightstep |
| **APM** | OpenTelemetry + Pyroscope | Datadog, Dynatrace |
| **Alerting** | Alertmanager + Karma | PagerDuty, Opsgenie |

**Стандартный бесплатный стек 2026:** OTel Collector + Prometheus + Loki + Tempo + Grafana.

---

## 9. Ingress / Gateway

| | NGINX Ingress | Traefik | Envoy Gateway | Cilium Gateway | Istio Gateway |
|---|---------------|---------|---------------|----------------|---------------|
| **Gateway API** | ✅ | ✅ | ✅ native | ✅ native | ✅ |
| **L7** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **mTLS** | ⚠️ | ✅ | ✅ | ✅ (eBPF) | ✅ |
| **Простота** | 🟢 | 🟢 | 🟡 | 🟡 | 🔴 |
| **Рекомендация** | Default | Auto-discovery | Heavy traffic | If on Cilium | If on Istio |

---

## 10. Secrets management

| | Sealed Secrets | External Secrets | Vault | SOPS |
|---|----------------|------------------|-------|------|
| **Где хранится** | В git (encrypted) | External (AWS SM, Vault) | Vault server | В git (encrypted) |
| **Rotation** | Мануал | Авто | Авто | Мануал |
| **K8s-native** | ✅ | ✅ | ⚠️ | ⚠️ |
| **Сложность** | 🟢 | 🟡 | 🔴 | 🟢 |
| **Когда** | Small team | Multi-cloud | Enterprise | GitOps + KMS |

---

## 11. Image build tools

| | Docker buildx | BuildKit standalone | Kaniko | Buildah | ko (Go) |
|---|---------------|---------------------|--------|---------|---------|
| **Daemon** | ✅ | ✅ | ❌ | ❌ | ❌ |
| **k8s-friendly** | ⚠️ (DinD) | ✅ | ✅ (native) | ✅ | ✅ |
| **Dockerfile** | ✅ | ✅ | ✅ | ✅ | ❌ (Go only) |
| **Multi-arch** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Speed** | ⚡⚡⚡ | ⚡⚡⚡ | ⚡⚡ | ⚡⚡ | ⚡⚡⚡⚡ |
| **Когда** | Local dev | CI standalone | k8s CI | Rootless | Go projects |

---

## 12. Policy engines

| | Kyverno | OPA Gatekeeper | jsPolicy |
|---|---------|----------------|----------|
| **Язык** | YAML | Rego | JavaScript/TS |
| **Кривая** | 🟢 | 🔴 | 🟡 |
| **Mutation** | ✅ | ✅ | ✅ |
| **Generate** | ✅ | ❌ | ✅ |
| **Рекомендация** | ✅ Default 2026 | Rego experts | JS shops |

---

## 13. Helm вс Kustomize вс Timoni вс CUE

| | Helm | Kustomize | Timoni | CUE |
|---|------|-----------|--------|-----|
| **Язык** | Go templates + YAML | YAML patches | CUE | CUE |
| **Типизация** | ❌ | ❌ | ✅ | ✅ |
| **Schema validation** | ⚠️ (values.schema.json) | ❌ | ✅ | ✅ |
| **Ecosystem** | 🌟 Огромный | Нативный | 🌱 Растет | 🌱 |
| **Когда** | Default ecosystem | Patches over Helm | Type-safe | Config as code |

---

## 🎯 TL;DR для старта в 2026

Этот стек работает в prod и полностью бесплатен:

```
Build:     docker buildx + BuildKit
Base:      distroless / chainguard
Registry:  GHCR
Local k8s: kind / k3d
CI/CD:     GitHub Actions
GitOps:    Argo CD
Scan:      Trivy + Syft + Cosign
Policy:    Kyverno
Mesh:      Linkerd (или Cilium Service Mesh)
Ingress:   Gateway API + Envoy Gateway
Secrets:   External Secrets Operator
Obs:       OTel + Prometheus + Loki + Tempo + Grafana
```

> 🔗 [README](README.md) • [MAP](MAP.md) • [ARCHITECTURE](ARCHITECTURE.md) • [DIAGRAMS](DIAGRAMS.md)
