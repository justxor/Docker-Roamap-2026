# ☸️ Этап 06. Orchestration — Kubernetes, Helm, ArgoCD

> **Цель:** уметь деплоить приложение в Kubernetes, упаковывать в Helm-чарты, настраивать GitOps через ArgoCD.

---

## ⏱️ Длительность

14-21 день

## 📋 Чеклист готовности

- [ ] Понимаю архитектуру k8s (control plane, nodes, etcd, kubelet, kube-proxy)
- [ ] Знаю основные объекты: Pod, Deployment, Service, Ingress, ConfigMap, Secret
- [ ] Знаю продвинутые: StatefulSet, DaemonSet, Job, CronJob, HPA/VPA/PDB
- [ ] Умею писать Helm-чарт с values для dev/staging/prod
- [ ] Знаю Kustomize и когда что использовать
- [ ] Настроил GitOps через ArgoCD
- [ ] Понимаю NetworkPolicy, RBAC, ServiceAccount

---

## 1. Локальный k8s

Не нужен production-кластер чтобы учиться:

| Решение | Когда |
|---------|-------|
| **kind** | Самое популярное, multi-node на docker |
| **k3d** | k3s в Docker, очень быстрый |
| **minikube** | Старичок, single-node |
| **Docker Desktop** | Включается чекбоксом |
| **colima** | macOS, легковесная VM |

```bash
# k3d (рекомендую)
brew install k3d
k3d cluster create dev --servers 1 --agents 2 -p "8080:80@loadbalancer"
kubectl get nodes
```

## 2. Pod — минимальная единица

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: app
      image: ghcr.io/user/app:v1
      resources:
        requests: { cpu: 100m, memory: 128Mi }
        limits:   { cpu: 500m, memory: 512Mi }
      ports:
        - containerPort: 8000
      livenessProbe:
        httpGet: { path: /health, port: 8000 }
        initialDelaySeconds: 10
        periodSeconds: 10
      readinessProbe:
        httpGet: { path: /ready, port: 8000 }
        periodSeconds: 5
      startupProbe:
        httpGet: { path: /health, port: 8000 }
        failureThreshold: 30
        periodSeconds: 2
```

**Probes:**
- **liveness** — жив ли процесс. Failure → restart.
- **readiness** — готов ли принимать трафик. Failure → убирается из Service endpoints.
- **startup** — даёт время на медленный старт перед включением liveness.

## 3. Deployment — управление набором Pod-ов

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0      # zero-downtime
  selector:
    matchLabels: { app: myapp }
  template:
    metadata:
      labels: { app: myapp }
    spec:
      containers:
        - name: app
          image: ghcr.io/user/app:v1
```

## 4. Service — стабильный endpoint

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: ClusterIP            # или NodePort / LoadBalancer
  selector: { app: myapp }
  ports:
    - port: 80
      targetPort: 8000
```

```bash
# Изнутри кластера:
curl http://myapp.<namespace>.svc.cluster.local
```

## 5. Ingress — HTTP-маршрутизация снаружи

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts: [api.example.com]
      secretName: api-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: myapp, port: { number: 80 } }
```

В 2026 многие переходят на **Gateway API** — современная замена Ingress.

## 6. ConfigMap и Secret

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: myapp-config }
data:
  LOG_LEVEL: "INFO"
  DATABASE_HOST: "postgres.db.svc.cluster.local"
---
apiVersion: v1
kind: Secret
metadata: { name: myapp-secrets }
type: Opaque
stringData:
  DB_PASSWORD: "secret"
```

**Никогда не клади Secrets в git!** Используй:
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [External Secrets Operator](https://external-secrets.io/) с AWS/GCP Secret Manager, Vault
- [SOPS](https://github.com/getsops/sops) с age/PGP

## 7. HPA — автомасштабирование

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: myapp }
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
```

Для custom metrics — **KEDA** (Kafka lag, RabbitMQ, Prometheus query, и т.д.).

## 8. PDB — Pod Disruption Budget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: myapp }
spec:
  minAvailable: 2
  selector: { matchLabels: { app: myapp } }
```
Защищает от drain ноды, который убил бы все Pod-ы сразу.

## 9. NetworkPolicy — firewall в k8s

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: myapp-deny-all }
spec:
  podSelector: { matchLabels: { app: myapp } }
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - podSelector: { matchLabels: { app: gateway } }
      ports: [{ protocol: TCP, port: 8000 }]
  egress:
    - to:
        - podSelector: { matchLabels: { app: postgres } }
      ports: [{ protocol: TCP, port: 5432 }]
```

Требует CNI с поддержкой NetworkPolicy (Calico, Cilium).

## 10. Helm — package manager

```bash
helm create myapp                           # скелет чарта
helm install myapp ./myapp -n prod          # установка
helm upgrade myapp ./myapp -f values.prod.yaml
helm uninstall myapp -n prod
helm list -A
helm rollback myapp 1
```

Структура чарта:
```
myapp/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── _helpers.tpl
│   └── tests/
│       └── test-connection.yaml
└── charts/                # subcharts
```

Шаблоны используют Go templates + Sprig:
```yaml
replicas: {{ .Values.replicaCount | default 3 }}
image: {{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}
{{- if .Values.ingress.enabled }}
# ...
{{- end }}
```

## 11. Kustomize — без шаблонов

```
base/
├── kustomization.yaml
├── deployment.yaml
└── service.yaml
overlays/
├── dev/
│   ├── kustomization.yaml
│   └── replica-patch.yaml
└── prod/
    └── kustomization.yaml
```

`kubectl apply -k overlays/prod` — применяет base + patches.

**Helm vs Kustomize:** Helm = шаблоны и values, Kustomize = layering patches. Можно совмещать.

## 12. ArgoCD — GitOps

GitOps: единственный источник правды — git-репо. ArgoCD синхронизирует git → cluster.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/user/k8s-manifests
    targetRevision: main
    path: apps/myapp/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Альтернатива: **Flux CD**.

## 13. Операторы — расширение API

CRD + Controller = собственный объект k8s.

Примеры в production:
- **cert-manager** — автоматический TLS через Let's Encrypt
- **Prometheus Operator** — управление мониторингом
- **PostgreSQL Operator (CNPG)** — PostgreSQL-кластеры
- **External Secrets Operator** — секреты из Vault/AWS

## 14. Чеклист production-Deployment

- [ ] resources: requests/limits
- [ ] probes: liveness + readiness + startup
- [ ] securityContext: runAsNonRoot, readOnlyRootFs, capabilities drop
- [ ] image: pinned by digest (не `:latest`)
- [ ] imagePullPolicy: IfNotPresent
- [ ] terminationGracePeriodSeconds
- [ ] strategy: RollingUpdate с maxUnavailable: 0
- [ ] affinity / topologySpreadConstraints для HA
- [ ] PDB
- [ ] HPA
- [ ] NetworkPolicy
- [ ] ServiceMonitor (Prometheus)
- [ ] PodMonitor / OTel
- [ ] Labels по best practice (app.kubernetes.io/*)

---

## 🧪 Практика

### Упражнение 1: kind-кластер
Подними kind или k3d. Задеплой простое приложение через kubectl. Проверь scale, restart, logs.

### Упражнение 2: Helm-чарт
Напиши Helm-чарт для своего приложения. С values для dev/staging/prod. Сделай `helm install` + `helm upgrade` + `helm rollback`.

### Упражнение 3: Ingress + TLS
Подними nginx-ingress + cert-manager. Получи Let's Encrypt сертификат на свой домен.

### Упражнение 4: HPA
Загрузи приложение через `hey` или `vegeta`. Убедись что HPA масштабирует.

### Упражнение 5: ArgoCD
Установи ArgoCD в кластер. Создай Application указывающий на свой git-репо. Сделай push → дождись авто-sync.

### Упражнение 6: NetworkPolicy
Напиши deny-all + allow конкретно. Проверь `kubectl exec` curl-ом.

---

## 📚 Ресурсы

### Telegram
- 🔥 [ai_machinelearning_big_data](https://t.me/ai_machinelearning_big_data)
- 🐍 [pythonl](https://t.me/pythonl)
- 📚 [Папка каналов](https://t.me/addlist/8vDUwYRGujRmZjFi)

### Документация
- [kubernetes.io/docs](https://kubernetes.io/docs/)
- [helm.sh/docs](https://helm.sh/docs/)
- [argo-cd.readthedocs.io](https://argo-cd.readthedocs.io/)
- [kustomize.io](https://kustomize.io/)

### Видео
- [TechWorld with Nana — Kubernetes Course](https://www.youtube.com/watch?v=X48VuDVv0do)
- [DevOps Toolkit — Viktor Farcic](https://www.youtube.com/@DevOpsToolkit)

### Книги (фрагменты)
- [Kubernetes Up & Running](https://www.oreilly.com/library/view/kubernetes-up-and/9781098110192/)
- [The Kubernetes Book — Nigel Poulton](https://nigelpoulton.com/books/)

### Tools
- [k9s](https://k9scli.io/) — TUI
- [lens](https://k8slens.dev/) — GUI
- [stern](https://github.com/stern/stern) — multi-pod tail
- [kubectx + kubens](https://github.com/ahmetb/kubectx)

---

## ➡️ Что дальше

[Этап 07 — Production](07-production.md): CI/CD, observability, troubleshooting.

[⬅ MAP](../MAP.md) · [⬅ README](../README.md)
