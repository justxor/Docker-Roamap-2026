# 🔍 Prompt: дебаг падающего контейнера

Используй когда контейнер падает в CrashLoopBackOff / OOMKilled / ImagePullBackOff / Pending.

---

## Промпт

Ты — SRE с опытом дебага production-k8s. Помоги найти и исправить проблему.

### Что произошло

<опиши симптомы: когда началось, что менялось, какие сервисы затронуты>

### Логи (`kubectl logs <pod> --previous`)
```
<вставь логи>
```

### Describe (`kubectl describe pod <pod>`)
```
<вставь Events, Last State, Conditions>
```

### Манифест
```yaml
<вставь Deployment/Pod YAML>
```

### Системные events (`kubectl get events --sort-by=.lastTimestamp`)
```
<вставь>
```

---

## Что я жду в ответе

1. **Root cause** — в 1-2 предложениях
2. **Доказательства** — какая строка в логах / манифесте это показывает
3. **Fix** — конкретный патч манифеста / Dockerfile / кода
4. **Проверка** — как верифицировать что исправлено (команда kubectl/curl)
5. **Превентивные меры** — чтобы не повторилось (probe, limit, retry, monitoring)

Популярные причины по симптомам:
- **CrashLoopBackOff** → падение на старте, liveness толстая, OOM, missing config
- **OOMKilled** → limits низкие, утечка памяти, GC issues
- **ImagePullBackOff** → неверный тег, private registry без imagePullSecret, rate limit
- **Pending** → нет node с нужным CPU/RAM, taint/toleration, PVC pending
- **502/503 в Ingress** → readiness не проходит, endpoint пустой, неверный service selector

---

- [stages/07-production.md](../stages/07-production.md)
