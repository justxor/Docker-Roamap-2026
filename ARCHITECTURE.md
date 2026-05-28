# 🏛️ Архитектура контейнерного мира

> Глубокое погружение в устройство Docker, containerd, runc и Kubernetes. Все диаграммы — Mermaid, рендерятся в GitHub нативно.

---

## 1. Полный стек: от kubectl до Linux Kernel

```mermaid
flowchart TB
    subgraph K8S[☸️ Kubernetes Control Plane]
        API[kube-apiserver]
        SCH[scheduler]
        CM[controller-manager]
        ETC[(etcd)]
    end
    subgraph NODE[🖥️ Worker Node]
        KL[kubelet]
        KP[kube-proxy]
        subgraph CRI[CRI Runtime]
            CTD[containerd]
            SHIM[containerd-shim-runc-v2]
        end
        subgraph OCI[OCI Runtime]
            RUNC[runc / crun]
        end
        subgraph LX[🐧 Linux Kernel]
            NS[namespaces]
            CG[cgroups v2]
            LSM[LSM: AppArmor/SELinux]
            SEC[seccomp-bpf]
            CAP[capabilities]
        end
    end
    USER[👤 kubectl apply] --> API
    API <--> ETC
    API --> SCH
    API --> CM
    SCH --> KL
    KL -->|CRI gRPC| CTD
    CTD --> SHIM
    SHIM --> RUNC
    RUNC --> NS
    RUNC --> CG
    RUNC --> LSM
    RUNC --> SEC
    RUNC --> CAP
    KP -.->|iptables/IPVS/eBPF| LX
```

---

## 2. Linux namespaces — изоляция

```mermaid
flowchart LR
    subgraph C[Контейнер = набор namespaces]
        PID[pid<br/>свой PID 1]
        NET[net<br/>свой eth0, iptables]
        MNT[mnt<br/>свой rootfs]
        UTS[uts<br/>свой hostname]
        IPC[ipc<br/>свой shared memory]
        USR[user<br/>UID mapping]
        CGR[cgroup<br/>свой /sys/fs/cgroup]
        TIM[time<br/>свой CLOCK_MONOTONIC]
    end
    PROC[Процесс контейнера] --> C
```

> Проверка: `lsns -p $(pgrep -f myapp)` показывает все namespaces процесса.

---

## 3. cgroups v2 — лимиты ресурсов

```mermaid
flowchart TB
    ROOT[/sys/fs/cgroup/] --> SYS[system.slice]
    ROOT --> USER[user.slice]
    ROOT --> DOCKER[docker/]
    DOCKER --> C1[abc123.../<br/>cpu.max=100000 100000<br/>memory.max=512M]
    DOCKER --> C2[def456.../<br/>cpu.max=50000 100000<br/>memory.max=1G]
    C1 --> P1[процесс контейнера #1]
    C2 --> P2[процесс контейнера #2]
```

Контроллеры: `cpu`, `memory`, `io`, `pids`, `hugetlb`, `rdma`, `misc`.

---

## 4. Жизненный цикл сборки образа

```mermaid
sequenceDiagram
    participant U as User
    participant CLI as docker CLI
    participant D as dockerd
    participant BK as BuildKit
    participant CTD as containerd
    participant R as Registry
    U->>CLI: docker buildx build --push -t app:v1 .
    CLI->>D: HTTP /build
    D->>BK: solve LLB graph
    BK->>BK: parse Dockerfile -> LLB
    BK->>BK: execute stages (parallel)
    BK->>BK: cache mounts + secrets
    BK->>CTD: snapshot layers
    CTD-->>BK: image manifest
    BK->>R: push layers (only new)
    R-->>BK: 201 Created
    BK-->>D: digest sha256:...
    D-->>CLI: build success
    CLI-->>U: ✅ pushed
```

---

## 5. Multi-stage build — оптимизация

```mermaid
flowchart LR
    subgraph S1[Stage: builder]
        B1[FROM python:3.13 AS builder]
        B2[pip install --target /deps]
        B3[COPY src /app]
    end
    subgraph S2[Stage: runtime]
        R1[FROM distroless/python3]
        R2[COPY --from=builder /deps /deps]
        R3[COPY --from=builder /app /app]
        R4[USER 10001]
        R5[ENTRYPOINT]
    end
    S1 -.отбрасывается.-> X[🗑️]
    S2 --> FINAL[📦 final image<br/>~50 MB вместо 1 GB]
```

---

## 6. Сеть Docker — bridge режим

```mermaid
flowchart LR
    subgraph HOST[🖥️ Docker Host]
        ETH0[eth0<br/>192.168.1.10]
        IPT[iptables<br/>nat + filter]
        DB[docker0<br/>172.17.0.1/16]
        subgraph C1[Container A]
            VE1[veth → eth0<br/>172.17.0.2]
        end
        subgraph C2[Container B]
            VE2[veth → eth0<br/>172.17.0.3]
        end
    end
    EXT[🌐 Internet] <-->|MASQUERADE| ETH0
    ETH0 <-->|DNAT -p 8080| IPT
    IPT --> DB
    DB --> VE1
    DB --> VE2
```

---

## 7. Storage drivers и слои

```mermaid
flowchart TB
    subgraph IMG[📦 Образ - read-only]
        L1[Layer 1: base FS]
        L2[Layer 2: apt install]
        L3[Layer 3: pip install]
        L4[Layer 4: COPY app]
    end
    subgraph CTR[🟢 Контейнер]
        RW[Read-Write layer<br/>copy-on-write]
    end
    L1 --> L2 --> L3 --> L4
    L4 --> RW
    RW --> OVL[overlayfs union mount<br/>/var/lib/docker/overlay2/]
```

---

## 8. Kubernetes Pod lifecycle

```mermaid
stateDiagram-v2
    [*] --> Pending: kubectl apply
    Pending --> ContainerCreating: scheduler assigned node
    ContainerCreating --> Running: image pulled, started
    Running --> Succeeded: exit 0 (Job)
    Running --> Failed: exit != 0
    Running --> Terminating: kubectl delete
    Terminating --> [*]: SIGTERM → grace → SIGKILL
    Failed --> CrashLoopBackOff: restartPolicy=Always
    CrashLoopBackOff --> Running: backoff timer
    ContainerCreating --> ImagePullBackOff: bad registry/creds
    ImagePullBackOff --> ContainerCreating: retry
```

---

## 9. Service discovery в Kubernetes

```mermaid
flowchart LR
    POD1[Pod app-1<br/>10.0.1.5] --> EP[(Endpoints)]
    POD2[Pod app-2<br/>10.0.1.6] --> EP
    POD3[Pod app-3<br/>10.0.1.7] --> EP
    EP --> SVC[Service app<br/>ClusterIP 10.96.0.10]
    SVC --> KP[kube-proxy<br/>iptables/IPVS/eBPF]
    KP --> POD1
    KP --> POD2
    KP --> POD3
    DNS[CoreDNS<br/>app.default.svc.cluster.local] --> SVC
    CLIENT[Клиент в кластере] --> DNS
```

---

## 10. GitOps deployment flow (ArgoCD)

```mermaid
sequenceDiagram
    participant DEV as Developer
    participant APP as app-repo
    participant CI as GitHub Actions
    participant REG as Registry
    participant CFG as config-repo (manifests)
    participant ARGO as ArgoCD
    participant K8S as Kubernetes
    DEV->>APP: git push
    APP->>CI: trigger workflow
    CI->>CI: test + build + scan + sign
    CI->>REG: docker push v1.2.3
    CI->>CFG: bump image tag (PR)
    CFG-->>CI: merge to main
    ARGO->>CFG: poll (3 min) / webhook
    ARGO->>K8S: kubectl apply / diff
    K8S-->>ARGO: status: Synced/Healthy
    ARGO-->>DEV: notify (Slack/UI)
```

---

## 11. Observability: три столпа

```mermaid
flowchart TB
    APP[Приложение<br/>OTel SDK] --> COL[OTel Collector]
    COL --> M[(Metrics<br/>Prometheus)]
    COL --> L[(Logs<br/>Loki)]
    COL --> T[(Traces<br/>Tempo/Jaeger)]
    M --> GRAF[📊 Grafana]
    L --> GRAF
    T --> GRAF
    M --> AM[Alertmanager]
    AM --> SLACK[🔔 Slack/PagerDuty]
```

---

## 12. Supply chain security — SLSA

```mermaid
flowchart LR
    SRC[📝 Source<br/>git commit signed] --> BUILD
    BUILD[🔨 Build<br/>BuildKit + hermetic] --> ART
    ART[📦 Artifact<br/>image + SBOM] --> SIGN
    SIGN[✍️ Sign<br/>Cosign keyless OIDC] --> ATT
    ATT[📜 Attest<br/>provenance + vuln scan] --> REG
    REG[(🏛️ Registry<br/>Harbor/GHCR)] --> VERIFY
    VERIFY[🔍 Verify<br/>Kyverno admission] --> DEPLOY
    DEPLOY[🚀 Deploy<br/>k8s]
```

---

## 13. Traffic flow в production

```mermaid
flowchart LR
    USER[👤 User] --> CDN[☁️ CDN<br/>Cloudflare]
    CDN --> LB[⚖️ Cloud LB<br/>L4]
    LB --> ING[🚪 Ingress<br/>nginx/Traefik/Gateway API]
    ING --> SVC1[Service A]
    ING --> SVC2[Service B]
    SVC1 --> POD1[Pods × N<br/>HPA scaled]
    SVC2 --> POD2[Pods × M]
    POD1 --> DB[(PostgreSQL<br/>StatefulSet)]
    POD2 --> CACHE[(Redis<br/>Sentinel)]
    POD1 --> Q[(Kafka<br/>Strimzi)]
    POD2 --> Q
```

---

## 14. Сравнение runtime

| Runtime | Язык | Особенности | Когда выбрать |
|---------|------|-------------|---------------|
| **runc** | Go | Reference OCI | Дефолт, везде работает |
| **crun** | C | В 10× быстрее runc | HPC, быстрый старт |
| **youki** | Rust | Memory-safe | Безопасность |
| **gVisor** | Go | User-space kernel | Multi-tenant изоляция |
| **Kata** | Go | Lightweight VM | Hardware isolation |
| **Firecracker** | Rust | microVM | Serverless (AWS Lambda) |

---

## 15. Эволюция: от chroot до WASM

```mermaid
timeline
    title История контейнеризации
    1979 : chroot (Unix V7)
    2000 : FreeBSD Jails
    2008 : LXC (Linux Containers)
    2013 : Docker 0.1
    2014 : Kubernetes 0.1
    2015 : OCI создан
    2016 : containerd выделен
    2018 : k8s deprecates dockershim
    2020 : rootless контейнеры зрелые
    2022 : WASM в k8s (runwasi)
    2024 : eBPF-native networking (Cilium стандарт)
    2026 : SLSA Level 3 как baseline
```

---

> 🔗 [README](README.md) · [MAP](MAP.md) · [stages/](stages/)
