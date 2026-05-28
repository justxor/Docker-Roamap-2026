# 📊 Диаграммы и схемы

> Коллекция детальных диаграмм для визуального понимания. Дополняет [MAP.md](MAP.md) и [ARCHITECTURE.md](ARCHITECTURE.md).

## 📑 Содержание

1. [Dockerfile anatomy](#1-dockerfile-anatomy)
2. [BuildKit graph](#2-buildkit-llb-graph)
3. [Compose dependency tree](#3-compose-dependency-tree)
4. [Kubernetes objects map](#4-kubernetes-objects-relationship)
5. [Helm rendering pipeline](#5-helm-rendering-pipeline)
6. [CNI / Service Mesh](#6-cni--service-mesh)
7. [Storage: CSI](#7-storage-csi-flow)
8. [Autoscaling layers](#8-autoscaling-layers)
9. [Disaster recovery](#9-disaster-recovery)
10. [Cost optimization](#10-cost-optimization-loop)

---

## 1. Dockerfile anatomy

```mermaid
flowchart TB
    subgraph META[📝 Метаданные]
        SYN[# syntax=docker/dockerfile:1.7]
        ARG[ARG BASE_IMAGE=python:3.13-slim]
        LBL[LABEL org.opencontainers.image.*]
    end
    subgraph BUILD[🔨 Build stage]
        FROM1[FROM image AS builder]
        WD1[WORKDIR /build]
        RUN1[RUN --mount=type=cache,target=/root/.cache pip install]
        COPY1[COPY pyproject.toml uv.lock ./]
    end
    subgraph RUN[🚀 Runtime stage]
        FROM2[FROM distroless/python3-debian12]
        COPY2[COPY --from=builder /opt /opt]
        USR[USER 10001:10001]
        EXP[EXPOSE 8080]
        HC[HEALTHCHECK CMD ...]
        ENT[ENTRYPOINT exec form]
    end
    META --> BUILD --> RUN
```

**Правила:**
- Порядок инструкций = порядок слоёв. Самые стабильные — выше.
- `COPY pyproject.toml` раньше `COPY . .` — кэш не инвалидируется.
- BuildKit cache mounts — не попадают в образ.

---

## 2. BuildKit LLB graph

```mermaid
flowchart LR
    SRC[(Источники<br/>Dockerfile + context)] --> FE[Frontend<br/>parse Dockerfile]
    FE --> LLB[(LLB DAG<br/>low-level builder)]
    LLB --> S1[stage: builder]
    LLB --> S2[stage: runtime]
    S1 -.cache.-> CACHE[(BuildKit cache<br/>local / registry / s3)]
    S2 -.cache.-> CACHE
    S1 --> SOLVE[Solver<br/>parallel execution]
    S2 --> SOLVE
    SOLVE --> EXP[Exporter]
    EXP --> IMG[(OCI image)]
    EXP --> REG[(Registry push)]
    EXP --> TAR[(local tarball)]
```

---

## 3. Compose dependency tree

```mermaid
flowchart TB
    API[api<br/>depends_on: db healthy, cache started] --> DB[postgres<br/>healthcheck pg_isready]
    API --> CACHE[redis<br/>healthcheck PING]
    API --> OTEL[otel-collector]
    OTEL --> JAEGER[jaeger]
    OTEL --> PROM[prometheus]
    PROM --> GRAF[grafana]
    LOKI[loki] --> GRAF
    PROMTAIL[promtail<br/>scrape logs] --> LOKI
    classDef ext fill:#1f6feb,color:#fff
    class API ext
```

**Профили:**

| Profile | Сервисы | Команда |
|---------|----------|---------|
| `core` | api, db, cache | `docker compose --profile core up` |
| `observability` | otel, prom, grafana, loki | `--profile observability` |
| `dev-tools` | pgadmin, redis-commander | `--profile dev-tools` |

---

## 4. Kubernetes objects relationship

```mermaid
flowchart TB
    NS[Namespace] --> DEP[Deployment]
    DEP --> RS[ReplicaSet]
    RS --> POD[Pod × N]
    POD --> CTR[Container]
    NS --> SVC[Service]
    SVC -.selector.-> POD
    NS --> ING[Ingress / Gateway]
    ING --> SVC
    NS --> CM[ConfigMap]
    NS --> SEC[Secret]
    CM -.envFrom.-> POD
    SEC -.envFrom.-> POD
    NS --> PVC[PersistentVolumeClaim]
    PVC --> PV[(PersistentVolume)]
    PVC -.volumes.-> POD
    NS --> HPA[HorizontalPodAutoscaler]
    HPA -.scales.-> DEP
    NS --> PDB[PodDisruptionBudget]
    PDB -.protects.-> POD
    NS --> NP[NetworkPolicy]
    NP -.filters.-> POD
    NS --> SA[ServiceAccount]
    SA -.identity.-> POD
    SA --> RB[RoleBinding] --> ROLE[Role]
```

---

## 5. Helm rendering pipeline

```mermaid
flowchart LR
    CHART[Chart.yaml + templates/] --> RENDER
    VAL[values.yaml] --> RENDER
    OVR[--values prod.yaml<br/>--set image.tag=v2] --> RENDER
    RENDER[helm template engine<br/>Go templates + sprig] --> MANIFEST[k8s manifests YAML]
    MANIFEST --> HOOK[pre-install hooks]
    HOOK --> APPLY[kubectl apply]
    APPLY --> K8S[(API server)]
    APPLY --> REL[(release secret<br/>helm history)]
```

---

## 6. CNI / Service Mesh

```mermaid
flowchart TB
    subgraph POD[Pod]
        APP[app container]
        SC[sidecar: envoy/linkerd-proxy]
    end
    APP <--localhost--> SC
    SC <--mTLS--> SC2[remote sidecar]
    SC2 --> APP2[remote app]
    subgraph CTRL[Control Plane]
        ISTIO[istiod / linkerd-control]
    end
    ISTIO -.config.-> SC
    ISTIO -.config.-> SC2
    CNI[CNI: Cilium / Calico / eBPF] -.pod network.-> POD
```

**Сравнение CNI:**

| CNI | Сильные стороны |
|-----|------------------|
| Cilium | eBPF, NetworkPolicy L7, observability (Hubble) |
| Calico | BGP, зрелый, NetworkPolicy |
| Flannel | Простой, для старта |
| Weave | Mesh, устаревает |

---

## 7. Storage CSI flow

```mermaid
sequenceDiagram
    participant U as User
    participant API as API server
    participant PVC as PVC
    participant SC as StorageClass
    participant CSI as CSI driver
    participant CLOUD as Cloud Storage
    participant POD as Pod
    U->>API: kubectl apply pvc.yaml
    API->>PVC: create
    PVC->>SC: lookup provisioner
    SC->>CSI: CreateVolume
    CSI->>CLOUD: API call (EBS/PD/Azure Disk)
    CLOUD-->>CSI: volume-id
    CSI-->>PVC: bind PV
    POD->>API: schedule with PVC
    API->>CSI: NodeStageVolume + NodePublishVolume
    CSI->>POD: mount /var/lib/kubelet/pods/.../
```

---

## 8. Autoscaling layers

```mermaid
flowchart TB
    APP[📈 Load increases] --> HPA[HPA<br/>scale pods 3 → 20]
    HPA --> VPA[VPA<br/>recommend CPU/RAM]
    HPA --> CA[Cluster Autoscaler /<br/>Karpenter]
    CA --> NEW[🆕 New nodes added]
    APP --> KEDA[KEDA<br/>scale on Kafka lag / SQS / cron]
    KEDA --> HPA
    NEW --> POD[New pods scheduled]
```

**Слои:**
- **HPA** — по CPU/RAM/custom metrics.
- **VPA** — подбор requests/limits.
- **KEDA** — event-driven (Kafka, RabbitMQ, Prometheus query).
- **Cluster Autoscaler / Karpenter** — новые ноды.

---

## 9. Disaster recovery

```mermaid
flowchart LR
    PRD[🏢 Production cluster] -->|Velero backup| S3[(S3 / GCS<br/>encrypted)]
    S3 --> DR[🆘 DR cluster<br/>another region]
    PRD -.async replication.-> DBR[(DB replica)]
    DBR --> DR
    OPS[👨‍💻 On-call] -->|trigger failover| DNS[Route53 / Cloudflare]
    DNS --> DR
```

**Метрики:**
- **RTO** (Recovery Time Objective) — за какое время восстановим.
- **RPO** (Recovery Point Objective) — сколько данных потеряем.
- Цель: RTO ≤ 15 мин, RPO ≤ 5 мин для критичных сервисов.

---

## 10. Cost optimization loop

```mermaid
flowchart LR
    M[📊 Метрики<br/>Kubecost / OpenCost] --> A[🔍 Анализ<br/>top spenders]
    A --> O1[VPA recommend]
    A --> O2[Spot/Preemptible<br/>nodes]
    A --> O3[Right-sizing<br/>requests/limits]
    A --> O4[Karpenter<br/>bin-packing]
    O1 --> R[🎉 -30% счёт]
    O2 --> R
    O3 --> R
    O4 --> R
    R --> M
```

---

## 🔗 Связанные документы

- [README](README.md) — входная точка
- [MAP](MAP.md) — общий маршрут
- [ARCHITECTURE](ARCHITECTURE.md) — глубокая архитектура
- [stages/](stages/) — этапы 00-07
