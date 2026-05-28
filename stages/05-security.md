# 🔐 Этап 05. Security — Supply Chain, SBOM, Signing

> **Цель:** научиться строить безопасные контейнеры: от non-root и distroless до подписи образов, SBOM и runtime-защиты.

---

## ⏱️ Длительность

5-7 дней

## 📋 Чеклист готовности

- [ ] Знаю принцип least privilege (non-root, read-only, cap-drop)
- [ ] Сканирую образы (Trivy / Grype)
- [ ] Генерирую SBOM (Syft)
- [ ] Подписываю образы (Cosign, keyless)
- [ ] Знаю SLSA уровни и provenance
- [ ] Применяю Pod Security Standards в k8s
- [ ] Понимаю runtime-защиту (Falco, Tetragon)

---

## 1. Принципы безопасности контейнера

### Non-root
```dockerfile
RUN groupadd -r app && useradd -r -g app -u 10001 app
USER app
```
**Никогда не запускай как root в production.** Если CVE в приложении → атакующий получает root в контейнере → ближе к побегу.

### Read-only root filesystem
```bash
docker run --read-only --tmpfs /tmp nginx
```
```yaml
# k8s
securityContext:
  readOnlyRootFilesystem: true
```
Атакующий не может писать в ФС → не может закрепиться.

### Drop capabilities
```yaml
securityContext:
  capabilities:
    drop: ["ALL"]
    add: ["NET_BIND_SERVICE"]  # только если нужно слушать порт <1024
```

### No new privileges
```bash
docker run --security-opt no-new-privileges nginx
```
Блокирует setuid-escalation.

### Seccomp / AppArmor / SELinux
```bash
docker run --security-opt seccomp=profile.json
docker run --security-opt apparmor=docker-default
```

## 2. Минимизация атак

| База | Размер | CVE-surface |
|------|--------|-------------|
| ubuntu:24.04 | 80 MB | много |
| debian:12-slim | 75 MB | средне |
| alpine:3.20 | 7 MB | мало (musl) |
| distroless/static | 2 MB | минимум |
| chainguard/static | 2 MB | минимум + 0-CVE policy |

**Правило:** меньше пакетов = меньше CVE.

## 3. Scanning — Trivy / Grype

```bash
# Trivy
trivy image nginx:1.27
trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:latest
trivy fs .                       # сканировать репо
trivy config Dockerfile          # IaC scan

# Grype (от Anchore)
grype nginx:1.27
```

В CI:
```yaml
# GitHub Actions
- uses: aquasecurity/trivy-action@master
  with:
    image-ref: ghcr.io/user/app:\${{ github.sha }}
    severity: HIGH,CRITICAL
    exit-code: '1'
```

## 4. SBOM — Software Bill of Materials

Список всех компонентов в образе. Стандарты: **SPDX**, **CycloneDX**.

```bash
# Syft
syft ghcr.io/user/app:v1 -o spdx-json > sbom.spdx.json
syft ghcr.io/user/app:v1 -o cyclonedx-json > sbom.cdx.json

# Через BuildKit
docker buildx build --sbom=true --provenance=true -t myapp:v1 --push .
```

Зачем SBOM:
- В случае CVE можно мгновенно ответить: "у нас в production есть log4j 2.14? — НЕТ"
- Compliance (regulated industries)
- SLSA / executive order 14028

## 5. Signing — Cosign

```bash
# Keyless (OIDC через GitHub) — рекомендуется
cosign sign ghcr.io/user/app:v1.0.0

# С ключом
cosign generate-key-pair
cosign sign --key cosign.key ghcr.io/user/app:v1.0.0

# Верификация
cosign verify ghcr.io/user/app:v1.0.0 \
  --certificate-identity=https://github.com/user/repo/.github/workflows/release.yml@refs/heads/main \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com
```

В k8s — admission policy через Kyverno / Sigstore Policy Controller, которая допускает только подписанные образы.

## 6. SLSA — Supply chain Levels for Software Artifacts

| Level | Что значит |
|-------|------------|
| 1 | Есть build process |
| 2 | Build process versioned + хостится в trusted service |
| 3 | Build platform изолирован + non-falsifiable provenance |
| 4 | Two-party review + hermetic builds |

GitHub Actions через [slsa-github-generator](https://github.com/slsa-framework/slsa-github-generator) даёт SLSA Level 3 из коробки.

## 7. Provenance — откуда что собрано

```bash
docker buildx imagetools inspect ghcr.io/user/app:v1 --format '{{json .Provenance}}'
```

Provenance показывает: какой commit, какой Dockerfile, какие base images использовались.

## 8. Pod Security Standards (k8s)

3 уровня:
- **Privileged** — никаких ограничений (для системных DaemonSet)
- **Baseline** — минимум защиты
- **Restricted** — максимум (требуется в production)

```yaml
# namespace-метка
labels:
  pod-security.kubernetes.io/enforce: restricted
  pod-security.kubernetes.io/audit: restricted
  pod-security.kubernetes.io/warn: restricted
```

## 9. Policy as Code — Kyverno / OPA Gatekeeper

```yaml
# Kyverno: запрещаем :latest
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: enforce
  rules:
    - name: validate-image-tag
      match:
        any:
        - resources: { kinds: [Pod] }
      validate:
        message: "Tag :latest forbidden"
        pattern:
          spec:
            containers:
              - image: "!*:latest"
```

## 10. Runtime-защита

### Falco — детекция аномалий в runtime
```yaml
# Falco-правило
- rule: Shell in Container
  desc: Detect shell execution inside container
  condition: container.id != host and proc.name in (sh, bash, zsh)
  output: "Shell started in container (%proc.cmdline)"
  priority: WARNING
```

### Tetragon (eBPF)
Современнее Falco, eBPF без необходимости патчить ядро.

## 11. Чеклист production-безопасности

- [ ] USER не root, UID > 10000
- [ ] read-only root filesystem
- [ ] capabilities: drop ALL, add только нужное
- [ ] no-new-privileges
- [ ] Trivy в CI с failing на HIGH+CRITICAL
- [ ] SBOM генерируется и хранится
- [ ] Образы подписаны Cosign
- [ ] Provenance generation включена
- [ ] Pod Security: restricted
- [ ] NetworkPolicy ограничивает трафик
- [ ] Secret manager (External Secrets, Vault) вместо ENV-секретов
- [ ] Регулярный pen-test образов

---

## 🧪 Практика

### Упражнение 1: harden Dockerfile
Возьми свой Dockerfile. Добавь non-root, читай чеклист — выполни всё. Проверь `docker run --user 10001 --read-only`.

### Упражнение 2: trivy в CI
Подключи Trivy GitHub Action. Получи failure на caught CVE. Зафикси.

### Упражнение 3: cosign keyless
Подпиши свой образ keyless через GHA. Верифицируй `cosign verify`.

### Упражнение 4: SBOM
Сгенерируй SBOM через syft. Найди в нём конкретный пакет.

### Упражнение 5: Kyverno
Установи Kyverno в kind/k3d. Примень политику запрета `:latest`. Проверь что запрещает.

### Упражнение 6: Falco
Подними Falco в k3d. Запусти Pod, сделай `kubectl exec` → bash. Найди событие в Falco-логах.

---

## 📚 Ресурсы

### Telegram
- 🔥 [ai_machinelearning_big_data](https://t.me/ai_machinelearning_big_data)
- 🐍 [pythonl](https://t.me/pythonl)
- 📚 [Папка каналов](https://t.me/addlist/8vDUwYRGujRmZjFi)

### Книги (бесплатно)
- [Container Security — Liz Rice](https://info.aquasec.com/container-security-book)

### Документация
- [SLSA](https://slsa.dev/)
- [Sigstore / Cosign](https://docs.sigstore.dev/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)

### Tools
- [Trivy](https://github.com/aquasecurity/trivy)
- [Grype](https://github.com/anchore/grype)
- [Syft](https://github.com/anchore/syft)
- [Cosign](https://github.com/sigstore/cosign)
- [Falco](https://falco.org/)
- [Tetragon](https://github.com/cilium/tetragon)
- [Kyverno](https://kyverno.io/)
- [docker-bench-security](https://github.com/docker/docker-bench-security)

---

## ➡️ Что дальше

[Этап 06 — Orchestration](06-orchestration.md): Kubernetes, Helm, ArgoCD.

[⬅ MAP](../MAP.md) · [⬅ README](../README.md)
