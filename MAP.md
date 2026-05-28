# 🗺️ Карта Docker Roadmap 2026

> Визуальная карта пути от Linux-основ до production Kubernetes. Все диаграммы — на Mermaid (рендерятся прямо в GitHub).

---

## 🧭 Общий маршрут обучения

```mermaid
flowchart TD
    A[00. Prerequisites<br/>Linux • процессы • сеть • FS] --> B[01. Fundamentals<br/>OCI • runtime • image vs container]
    B --> C[02. Dockerfile<br/>multi-stage • BuildKit • cache]
    B --> D[03. Compose<br/>multi-service • healthchecks • profiles]
    B --> E[04. Network & Storage<br/>bridge/overlay • volumes • CSI]
    C --> F[05. Security<br/>SBOM • signing • scan • supply chain]
    D --> F
    E --> F
    F --> G[06. Orchestration<br/>Kubernetes • Helm • ArgoCD]
    G --> H[07. Production<br/>CI/CD • observability • SRE]
    H --> I{{🎯 Senior Container Engineer 2026}}
    classDef base fill:#0d1117,stroke:#58a6ff,color:#c9d1d9;
    classDef goal fill:#1f6feb,stroke:#58a6ff,color:#fff,font-weight:bold;
    class A,B,C,D,E,F,G,H base;
    class I goal;
```

---

## ⏱️ Таймлайн по неделям

```mermaid
gantt
    title Docker Roadmap 2026 — 16 недель
    dateFormat  YYYY-MM-DD
    axisFormat  W%V
    section Основы
    00. Prerequisites      :a1, 2026-01-01, 7d
    01. Fundamentals       :a2, after a1, 7d
    section Build
    02. Dockerfile         :b1, after a2, 14d
    03. Compose            :b2, after b1, 10d
    04. Network & Storage  :b3, after b2, 10d
    section Secure
    05. Security & Supply  :c1, after b3, 14d
    section Scale
    06. Orchestration k8s  :d1, after c1, 21d
    07. Production / SRE   :d2, after d1, 21d
    section Финал
    Capstone-проект        :crit, e1, after d2, 14d
```

---

## 🏗️ Архитектура контейнерного стека

```mermaid
flowchart TB
    subgraph User[👤 User Space]
        CLI[docker / nerdctl / podman CLI]
        Compose[docker compose]
    end
    subgraph Engine[🐳 Container Engine]
        D[dockerd / Docker Engine]
        BK[BuildKit]
    end
    subgraph Runtime[⚙️ Container Runtime]
        CTD[containerd]
        SHIM[containerd-shim-runc-v2]
        RUNC[runc / crun / youki]
    end
    subgraph Kernel[🐧 Linux Kernel]
        NS[namespaces<br/>pid/net/mnt/uts/ipc/user/cgroup/time]
        CG[cgroups v2]
        SEC[seccomp • AppArmor • SELinux • capabilities]
        OVL[overlayfs / btrfs / zfs]
    end
    CLI --> D
    Compose --> D
    D --> BK
    D --> CTD
    BK --> CTD
    CTD --> SHIM --> RUNC
    RUNC --> NS
    RUNC --> CG
    RUNC --> SEC
    RUNC --> OVL
```

---

## 🛤️ Треки по ролям

```mermaid
flowchart LR
    subgraph Backend[💻 Backend / App Developer]
        BE1[02. Dockerfile] --> BE2[03. Compose] --> BE3[Local dev: watch + hot reload]
    end
    subgraph DevOps[🛠️ DevOps / Platform]
        DO1[02-04 Build/Net/Storage] --> DO2[06. k8s + Helm] --> DO3[07. CI/CD + ArgoCD]
    end
    subgraph Sec[🔐 Security / DevSecOps]
        SE1[05. Security] --> SE2[SBOM + Cosign + SLSA] --> SE3[Kyverno + Falco]
    end
    subgraph SRE[📊 SRE / Platform Eng]
        SR1[06. k8s deep] --> SR2[Observability OTel] --> SR3[Capacity + cost]
    end
    subgraph ML[🧠 ML / MLOps]
        ML1[02. Dockerfile + GPU] --> ML2[Compose GPU stack] --> ML3[k8s + Kueue + KServe]
    end
```

---

## 📊 Сложность и приоритет

| Этап | Сложность | Приоритет | Время | Зависит от |
|------|-----------|-----------|-------|------------|
| 00. Prerequisites | 🟢 Easy | 🔴 Must | 1 нед | — |
| 01. Fundamentals | 🟢 Easy | 🔴 Must | 1 нед | 00 |
| 02. Dockerfile | 🟡 Medium | 🔴 Must | 2 нед | 01 |
| 03. Compose | 🟡 Medium | 🔴 Must | 1.5 нед | 02 |
| 04. Network & Storage | 🟡 Medium | 🟠 Should | 1.5 нед | 01 |
| 05. Security | 🟠 Hard | 🔴 Must | 2 нед | 02,03,04 |
| 06. Orchestration | 🔴 Hard+ | 🟠 Should | 3 нед | 05 |
| 07. Production | 🔴 Hard+ | 🟡 Could | 3 нед | 06 |

---

## 🔄 Lifecycle образа и контейнера

```mermaid
stateDiagram-v2
    [*] --> Dockerfile
    Dockerfile --> Build: docker build
    Build --> Image: layers + manifest
    Image --> Registry: docker push
    Registry --> Pull: docker pull
    Pull --> Created: docker create
    Created --> Running: docker start
    Running --> Paused: docker pause
    Paused --> Running: docker unpause
    Running --> Stopped: docker stop (SIGTERM)
    Running --> Killed: docker kill (SIGKILL)
    Stopped --> Running: docker start
    Stopped --> Removed: docker rm
    Killed --> Removed: docker rm
    Removed --> [*]
```

---

## 🧱 Многослойная структура образа

```mermaid
flowchart BT
    L0[Base layer: distroless / alpine / ubuntu] --> L1[OS deps: apt/apk install]
    L1 --> L2[Language runtime: python/node/jvm]
    L2 --> L3[Dependencies: pip/npm/maven]
    L3 --> L4[App code: COPY . .]
    L4 --> L5[Config & entrypoint]
    L5 --> RW[🟢 Read-Write Layer<br/>container runtime]
    style RW fill:#238636,stroke:#2ea043,color:#fff
    style L0 fill:#161b22,stroke:#30363d,color:#c9d1d9
```

---

## 🌐 Сетевые драйверы

```mermaid
flowchart LR
    Host[🖥️ Docker Host] --> B[bridge<br/>default, single-host]
    Host --> H[host<br/>shared host ns]
    Host --> N[none<br/>no network]
    Host --> O[overlay<br/>multi-host VXLAN]
    Host --> M[macvlan<br/>L2 VLAN]
    Host --> I[ipvlan<br/>L3]
    B --> UC1[локальная разработка]
    H --> UC2[максимум perf, ноль изоляции]
    O --> UC3[Swarm / multi-node]
    M --> UC4[legacy интеграция в LAN]
```

---

## 🔐 Слои безопасности (Defense-in-Depth)

```mermaid
flowchart TB
    subgraph L1[1. Image]
        I1[distroless / chainguard]
        I2[pinned digests]
        I3[Trivy / Grype scan]
        I4[SBOM Syft]
        I5[Cosign signature]
    end
    subgraph L2[2. Build]
        B1[BuildKit secrets]
        B2[SLSA provenance]
        B3[reproducible builds]
    end
    subgraph L3[3. Runtime]
        R1[non-root UID]
        R2[read-only fs]
        R3[drop ALL capabilities]
        R4[seccomp / AppArmor]
        R5[no-new-privileges]
    end
    subgraph L4[4. Orchestrator]
        O1[Pod Security restricted]
        O2[NetworkPolicy default-deny]
        O3[Kyverno / OPA]
        O4[Falco / Tetragon]
    end
    L1 --> L2 --> L3 --> L4
```

---

## 🚀 Production CI/CD Pipeline

```mermaid
flowchart LR
    G[git push] --> CI[GitHub Actions]
    CI --> L[lint hadolint]
    CI --> T[unit tests]
    L --> BK[buildx multi-arch]
    T --> BK
    BK --> SC[Trivy scan]
    BK --> SB[Syft SBOM]
    SC -->|pass| SG[Cosign sign keyless]
    SB --> AT[Cosign attest]
    SG --> REG[(GHCR / Harbor)]
    AT --> REG
    REG --> CD[ArgoCD / Flux]
    CD --> STG[Staging cluster]
    STG -->|smoke OK| PRD[Production canary]
    PRD --> OBS[OTel → Prometheus/Loki/Tempo]
```

---

## 📚 Где учить — приоритет источников

```mermaid
flowchart TD
    S1[1️⃣ Официальная документация<br/>docs.docker.com • kubernetes.io] --> S2
    S2[2️⃣ Спецификации<br/>OCI • CNCF • Compose Spec] --> S3
    S3[3️⃣ Telegram-каналы RU<br/>@ai_machinelearning_big_data • @pythonl] --> S4
    S4[4️⃣ Бесплатные курсы<br/>KodeKloud • Killercoda • Play with Docker] --> S5
    S5[5️⃣ YouTube<br/>TechWorld with Nana • Bret Fisher • DevOps Toolkit] --> S6
    S6[6️⃣ Книги (free)<br/>Docker Deep Dive • Kubernetes Up & Running] --> S7
    S7[7️⃣ Практика<br/>Capstone-проект на GitHub]
```

---

## 🎯 Capstone-проект (финальный артефакт)

Собрать в одном репозитории все навыки:

1. **Микросервис** (Python/Go/Node) с multi-stage Dockerfile на distroless.
2. **docker-compose.yml** с api+postgres+redis+otel-collector+jaeger+prometheus+grafana.
3. **Helm-чарт** с HPA, PDB, NetworkPolicy, restricted PodSecurity.
4. **GitHub Actions**: build → scan → sign → SBOM → attest → push → ArgoCD sync.
5. **Observability**: OTel-инструментирование + дашборд Grafana + алёрты.
6. **README** на русском, MIT-лицензия, скриншоты, ADR-документы.

```mermaid
mindmap
  root((Capstone 2026))
    Build
      multi-stage
      distroless
      buildx multi-arch
    Compose
      profiles
      watch
      healthchecks
    k8s
      Helm
      ArgoCD
      HPA + PDB
    Security
      Trivy
      Cosign
      SBOM
    Observability
      OTel
      Prometheus
      Loki
      Tempo
    Docs
      README RU
      ADR
      diagrams
```

---

## ✅ Чек-лист «готов к Senior»

- [ ] Понимаю разницу между namespaces, cgroups, capabilities.
- [ ] Пишу Dockerfile с multi-stage, BuildKit cache mounts, secrets.
- [ ] Знаю Compose v2 spec наизусть (profiles, watch, healthcheck).
- [ ] Запускаю Trivy + Syft + Cosign в CI.
- [ ] Деплою через Helm + ArgoCD с canary/blue-green.
- [ ] Настраиваю OTel → Prometheus/Loki/Tempo.
- [ ] Дебажу CrashLoopBackOff, OOMKilled, ImagePullBackOff вслепую.
- [ ] Снижаю размер образа в 5-10 раз через distroless / chainguard.
- [ ] Подписываю образы Cosign keyless (OIDC) и проверяю в admission.

---

> 🔗 Связанные документы: [README](README.md) • [stages/](stages/) • [templates/](templates/) • [prompts/](prompts/) • [cheatsheets/](cheatsheets/)
