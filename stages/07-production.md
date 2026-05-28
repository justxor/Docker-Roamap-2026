# 🚀 Этап 07. Production — CI/CD, Observability, Troubleshooting

> **Цель:** держать контейнеры в production. CI/CD pipelines, мониторинг, логи, трейсы, дебаг падающих Pod-ов.

---

## ⏱️ Длительность

7-14 дней

## 📋 Чеклист готовности

- [ ] CI/CD pipeline: build → test → scan → sign → push → deploy
- [ ] Multi-arch builds в CI
- [ ] Observability: metrics + logs + traces
- [ ] Алёрты в Alertmanager / Grafana / OnCall
- [ ] Умею дебажить CrashLoopBackOff, ImagePullBackOff, OOMKilled
- [ ] Знаю как делать canary / blue-green деплой
- [ ] Backup-стратегия для stateful workloads

---

## 1. CI/CD pipeline — GitHub Actions

```yaml
name: build-deploy

on:
  push:
    branches: [main]
    tags: ['v*']

permissions:
  contents: read
  packages: write
  id-token: write   # для cosign keyless

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: \${{ github.actor }}
          password: \${{ secrets.GITHUB_TOKEN }}

      - name: Metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/\${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha,prefix=sha-

      - name: Build & push
        id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: \${{ steps.meta.outputs.tags }}
          labels: \${{ steps.meta.outputs.labels }}
          provenance: true
          sbom: true
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/\${{ github.repository }}@\${{ steps.build.outputs.digest }}
          severity: HIGH,CRITICAL
          exit-code: '1'

      - name: Sign with Cosign
        env:
          DIGEST: \${{ steps.build.outputs.digest }}
          TAGS: \${{ steps.meta.outputs.tags }}
        run: |
          cosign sign --yes "ghcr.io/\${{ github.repository }}@\${DIGEST}"
```

## 2. Observability — три столпа

### Metrics (Prometheus)
```python
# приложение экспортит /metrics
from prometheus_client import Counter, Histogram, start_http_server
requests = Counter('http_requests_total', 'Total HTTP requests', ['method', 'endpoint', 'status'])
latency = Histogram('http_request_duration_seconds', 'Latency', ['endpoint'])
```

В k8s — ServiceMonitor для Prometheus Operator:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata: { name: myapp }
spec:
  selector: { matchLabels: { app: myapp } }
  endpoints:
    - port: metrics
      interval: 15s
```

### Logs (Loki / Vector / Fluent Bit)
Структурированный JSON-лог:
```json
{"ts": "2026-05-28T12:00:00Z", "level": "info", "service": "api", "trace_id": "abc", "msg": "request handled", "duration_ms": 12}
```

Loki индексирует labels, не содержимое — дешевле Elasticsearch.

### Traces (OpenTelemetry / Tempo / Jaeger)
```python
from opentelemetry import trace
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
FastAPIInstrumentor.instrument_app(app)
```

OTel — стандарт 2026, заменил Jaeger SDK и Zipkin SDK.

### Стек "LGTM"
- **L**oki — logs
- **G**rafana — UI
- **T**empo — traces
- **M**imir — long-term Prometheus

## 3. Алёрты

```yaml
# PrometheusRule
groups:
  - name: myapp
    rules:
      - alert: HighErrorRate
        expr: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels: { severity: critical }
        annotations:
          summary: "Error rate > 5% for 5min"

      - alert: PodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
```

Отправка: Alertmanager → PagerDuty / Slack / Telegram / Grafana OnCall (free).

## 4. Troubleshooting playbook

### CrashLoopBackOff
```bash
kubectl describe pod <pod>
kubectl logs <pod> --previous       # ← логи предыдущей попытки
kubectl get events --sort-by=.lastTimestamp
```

Частые причины:
- liveness probe слишком агрессивная
- OOMKilled (увеличь memory limit)
- падение на старте (читай logs --previous)
- entrypoint неправильный

### ImagePullBackOff
```bash
kubectl describe pod <pod>
# смотри Events → "Failed to pull image"
```
- неверное имя/тег
- private registry без imagePullSecret
- rate limit на Docker Hub (используй GHCR / pull-through cache)

### OOMKilled
```bash
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
# OOMKilled
```
- увеличь `resources.limits.memory`
- профилируй: `py-spy`, `pprof`, `heaptrack`

### Pending Pod
```bash
kubectl describe pod <pod>
# 0/3 nodes are available: ...
```
- нет нод с нужным CPU/RAM
- nodeSelector / affinity не матчатся
- PVC pending (нет storage class)

### Networking
```bash
# Pod не видит сервис
kubectl exec -it <pod> -- nslookup <svc-name>
kubectl exec -it <pod> -- curl -v http://<svc-name>:80

# NetworkPolicy блокирует?
kubectl get networkpolicy -A

# Pod в правильной сети?
kubectl get pod -o wide
```

### Debug-Pod (ephemeral container)
```bash
kubectl debug -it <pod> --image=nicolaka/netshoot --target=<container>
```
Для distroless / scratch образов — единственный способ зайти "внутрь".

## 5. Deployment стратегии

### RollingUpdate (по умолчанию)
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
```

### Blue-Green
Две версии параллельно, переключение через Service selector.

### Canary
- Через Service + два Deployment-а с разными labels + Ingress canary annotations
- Через **Flagger** / **Argo Rollouts** — автоматический canary с метриками

### Argo Rollouts пример
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 5m }
        - analysis:
            templates: [{ templateName: success-rate }]
        - setWeight: 50
        - pause: { duration: 10m }
        - setWeight: 100
```

## 6. Multi-arch registry

Используй GHCR для public, Harbor для self-hosted enterprise.

Pull-through cache:
```bash
# Harbor proxy cache → Docker Hub
# чтобы не ловить rate limit
```

## 7. Backup для stateful

- **Velero** — backup k8s + PVC
- **CNPG (Cloud Native PostgreSQL)** — встроенный backup в S3
- **Longhorn** — distributed storage с snapshots
- **Restic / Kopia** в CronJob — для произвольных volume

## 8. Cost optimization

- **Karpenter** (AWS) — авто-провижининг нод по нагрузке
- **kubecost** / **OpenCost** — учёт расходов
- **descheduler** — балансировка Pod-ов
- **VPA** — рекомендации по requests
- **Spot instances** для batch-нагрузки

## 9. Production-чеклист

- [ ] CI: lint + test + scan + sign + push
- [ ] CD: GitOps (ArgoCD / Flux)
- [ ] Прод-кластер ≥ 3 control plane, ≥ 3 worker
- [ ] Резервный регион / availability zones
- [ ] Backup-strategy документирована и тестируется
- [ ] DR-tests (восстановление кластера с нуля)
- [ ] Observability stack развёрнут
- [ ] Алёрты подключены к pager
- [ ] Runbook-и для каждого алёрта
- [ ] Capacity-planning (HPA + VPA + Cluster Autoscaler)
- [ ] Security: trivy, kyverno, falco, network policies
- [ ] Secret manager (не plain k8s Secrets)
- [ ] Регулярные образ-обновления и pin digest

---

## 🧪 Практика

### Упражнение 1: pipeline
Напиши полный GitHub Actions: build → trivy → SBOM → cosign → push в GHCR. Сделай так чтобы HIGH CVE падал билд.

### Упражнение 2: observability
Подними kube-prometheus-stack (Helm). Подключи свои Pod-ы через ServiceMonitor. Сделай дашборд в Grafana.

### Упражнение 3: алёрт
Напиши PrometheusRule на high error rate. Триггерни алёрт (нагрузи 5xx). Получи Slack/Telegram-уведомление.

### Упражнение 4: дебаг
Намеренно сломай Pod (неверный image, OOM, crash на старте). Дебагни через kubectl describe + logs --previous.

### Упражнение 5: canary
Разверни Argo Rollouts. Сделай canary с пошаговой раскаткой 10% → 50% → 100%.

### Упражнение 6: backup
Установи Velero. Сделай backup namespace, удали ресурсы, восстанови.

---

## 📚 Ресурсы

### Telegram
- 🔥 [ai_machinelearning_big_data](https://t.me/ai_machinelearning_big_data)
- 🐍 [pythonl](https://t.me/pythonl)
- 📚 [Папка каналов](https://t.me/addlist/8vDUwYRGujRmZjFi)

### Документация
- [docs.github.com/actions](https://docs.github.com/en/actions)
- [opentelemetry.io](https://opentelemetry.io/)
- [prometheus.io/docs](https://prometheus.io/docs/)
- [grafana.com/docs](https://grafana.com/docs/)
- [argoproj.github.io/rollouts](https://argoproj.github.io/argo-rollouts/)

### Видео
- [Bret Fisher — Docker in production](https://www.youtube.com/@BretFisher)
- [Sysdig — Falco demos](https://www.youtube.com/@Sysdig)

### Tools
- [Velero](https://velero.io/) — k8s backup
- [Karpenter](https://karpenter.sh/) — autoscaler
- [OpenCost](https://www.opencost.io/) — cost monitoring
- [k6](https://k6.io/) — load testing

---

## 🎓 Поздравляем!

Ты прошёл весь Docker Roadmap 2026. Дальше — собственные production-кластеры, контрибьюты в open source, изучение сложных тем: service mesh (Istio ambient, Linkerd, Cilium), eBPF, WebAssembly + Wasm runtime (containerd-wasm-shim).

[⬅ MAP](../MAP.md) · [⬅ README](../README.md)
