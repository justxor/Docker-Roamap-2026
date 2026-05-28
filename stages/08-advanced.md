# 08. Advanced — будущее контейнеров

> Продвинутые темы для тех, кто прошёл 00-07. eBPF, WASM, service mesh, multi-cluster, GPU.

## 🎯 Цели этапа

- Понять eBPF и почему это будущее networking/security.
- Запустить WASM-модуль в k8s через runwasi.
- Развернуть Istio/Linkerd и понять mTLS, traffic shifting.
- Разобраться в multi-cluster (Karmada, Cluster API).
- Запустить GPU workload в k8s (NVIDIA device plugin, MIG).

---

## 1. eBPF — программируемый ядро

**Что это:** безопасный VM внутри ядра Linux. Даёт возможность выполнять верифицированный код в ядре без patches.

**Где используется:**

| Проект | Сфера |
|---------|-------|
| **Cilium** | CNI, NetworkPolicy L7, load balancing |
| **Hubble** | Observability потоков |
| **Falco** | Runtime security (syscall мониторинг) |
| **Tetragon** | Security observability + enforcement |
| **Pixie** | Auto-instrumentation без кода |
| **Parca / Pyroscope** | Continuous profiling |

**Где учить (бесплатно):**
- [ebpf.io](https://ebpf.io/) — официальный хаб.
- [Cilium Labs](https://isovalent.com/labs/) — бесплатные интерактивные курсы.
- [Learning eBPF](https://github.com/lizrice/learning-ebpf) Liz Rice — сопроводительный код.

---

## 2. WASM в контейнерах

**Почему:** старт в миллисекунды, размер в Байтах/КБ, песочница по дизайну, cross-platform.

**Стек:**
- **runwasi** — OCI-shim для containerd, запуск WASM как контейнер.
- **WasmEdge / Wasmtime / Wasmer** — runtime.
- **SpinKube** — k8s-native WASM platform.

**Пример:**

```bash
# Build WASM module
cargo build --target wasm32-wasip1 --release

# Run via wasmtime
wasmtime target/wasm32-wasip1/release/app.wasm

# Or via containerd + runwasi
ctr run --runtime io.containerd.wasmtime.v1 \
  docker.io/library/app:wasm app
```

---

## 3. Service Mesh

**Сравнение:**

| | Istio | Linkerd | Cilium Service Mesh |
|---|-------|---------|---------------------|
| **Sidecar** | Envoy | linkerd2-proxy (Rust) | sidecarless (eBPF) |
| **Resources** | Тяжёлый | Лёгкий | Минимальные |
| **mTLS** | ✅ | ✅ (дефолт) | ✅ |
| **L7 policies** | ✅ богатые | ✅ простые | ✅ |
| **Кривая входа** | Высокая | Низкая | Средняя |
| **Для чего** | Enterprise | Startup-friendly | eBPF-native |

**Где учить:**
- [Istio docs](https://istio.io/latest/docs/) — официальная дока.
- [Linkerd workshop](https://linkerd.io/2/getting-started/) — 30 мин.
- [Solo.io Academy](https://academy.solo.io/) — бесплатные курсы Istio.

---

## 4. Multi-cluster

**Паттерны:**

1. **Active-passive** — prod + DR, DNS failover.
2. **Active-active** — traffic split по регионам.
3. **Cluster mesh** — Cilium ClusterMesh, Istio multi-primary.
4. **Hub-spoke** — Karmada, fleet management.

**Инструменты:**
- **Cluster API (CAPI)** — декларативное управление кластерами.
- **Karmada** — распределённый scheduler.
- **Fleet (Rancher)** — GitOps на флот кластеров.
- **Argo CD ApplicationSets** — развёртывание в N кластеров.

---

## 5. GPU и ML workloads

**Стек:**
- **NVIDIA Container Toolkit** — доступ к GPU в контейнерах.
- **NVIDIA Device Plugin** — GPU в k8s как ресурс `nvidia.com/gpu`.
- **MIG** (Multi-Instance GPU) — разделение A100/H100 на 7 виртуальных GPU.
- **Kueue** — batch-job queue.
- **KServe** — inference serving.
- **Volcano** — batch scheduler для AI/ML.

**Пример Pod с GPU:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  restartPolicy: Never
  containers:
    - name: cuda
      image: nvcr.io/nvidia/cuda:12.5.0-base-ubuntu24.04
      command: ["nvidia-smi"]
      resources:
        limits:
          nvidia.com/gpu: 1
```

---

## 6. Confidential Computing

**Суть:** контейнеры выполняются в изолированной аппаратной энклаве (Intel SGX, AMD SEV-SNP, Nitro Enclaves).

**Инструменты:**
- **Kata Containers** — lightweight VM по умолчанию.
- **Confidential Containers (CoCo)** — CNCF, k8s-native.
- **gVisor** — user-space ядро для изоляции.

---

## 7. Platform Engineering

**Паттерны:**
- **Internal Developer Platform (IDP)** — Backstage, Port.
- **Self-service** — Crossplane, kratix, KubeVela.
- **Templates** — cookiecutter, kustomize generators.

**Пример Backstage в минуту:**

```bash
npx @backstage/create-app@latest
cd my-backstage && yarn dev
```

---

## 8. FinOps для контейнеров

- **OpenCost / Kubecost** — cost allocation по namespace/label.
- **Karpenter** — spot-aware автомасштабирование.
- **Goldilocks** — рекомендации requests/limits.
- **kube-green** — sleep mode для dev-namespaces ночью.

---

## 📚 Ресурсы

- [CNCF Landscape](https://landscape.cncf.io/) — все cloud-native проекты.
- [Kubernetes Slack](https://kubernetes.slack.com/) — живое общение.
- Telegram: @ai_machinelearning_big_data, @pythonl, [папка](https://t.me/addlist/8vDUwYRGujRmZjFi).

## ✅ Чек-лист

- [ ] Запустил Cilium в kind, включил Hubble UI.
- [ ] Сбился WASM-модуль, запустил в wasmtime.
- [ ] Развернул Linkerd, настроил mTLS.
- [ ] Поднял два кластера + ArgoCD ApplicationSet.
- [ ] Посчитал стоимость namespace в OpenCost.

→ [Следующий этап: Capstone-проект](../MAP.md#-capstone-проект-финальный-артефакт)
