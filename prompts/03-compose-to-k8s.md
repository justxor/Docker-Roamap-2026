# 🔄 Prompt: конвертация docker-compose → Kubernetes + Helm

Используй когда нужно перенести dev compose-стек в production-k8s.

---

## Промпт

Ты — platform engineer с опытом миграции docker-compose приложений в Kubernetes. Конвертируй этот docker-compose.yml в k8s-манифесты + Helm chart.

Для каждого сервиса:
1. Создай Deployment / StatefulSet (выбери правильный тип)
2. Создай Service (ClusterIP по умолчанию, LoadBalancer/Ingress для внешних)
3. ConfigMap для environment variables
4. Secret для секретов (External Secrets Operator или Sealed Secrets)
5. PVC для volumes (выбери storage class)
6. healthcheck → livenessProbe + readinessProbe + startupProbe
7. depends_on → init containers или retry-логика в приложении
8. resource requests/limits (оцени по размеру image и типу сервиса)
9. SecurityContext: non-root, readOnlyRootFs, capabilities drop ALL
10. NetworkPolicy: ingress + egress правила
11. HPA если приложение stateless
12. PodDisruptionBudget

Дай:
- Отдельные YAML файлы в k8s/ (deployment.yaml, service.yaml, configmap.yaml, ...)
- Helm-чарт с values.yaml, templates/, _helpers.tpl
- README с инструкциями деплоя в dev/staging/prod
- Без костылей, только production-best-practices

---

## Мой docker-compose.yml

```yaml
<вставь>
```

## Контекст
- K8s версия: 1.30+
- Cloud / on-prem: <укажи>
- Ingress controller: <nginx/traefik/none>
- Storage class: <gp3/longhorn/...>

---

## См. также

- [stages/06-orchestration.md](../stages/06-orchestration.md)
- [templates/k8s-starter](../templates/k8s-starter/)
- [templates/helm-starter](../templates/helm-starter/)
