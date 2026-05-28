# 🔐 Prompt: аудит безопасности контейнера

Используй перед выкаткой в production или после CVE-incident.

---

## Промпт

Ты — security engineer с фокусом на container security. Проведи полный аудит по чеклисту:

### Image-level
1. Базовый образ pinned by digest?
2. Distroless / Chainguard используется?
3. Нет лишних пакетов (curl, wget, ssh, sudo, etc.)?
4. Trivy/Grype scan: HIGH/CRITICAL CVE = 0?
5. SBOM сгенерирован и сохранен?
6. Cosign подпись есть и верифицируема?
7. Provenance (SLSA) включён?
8. Секретов в layers нет (проверить trufflehog)?

### Runtime-level
9. Non-root user (UID > 10000)?
10. readOnlyRootFilesystem: true?
11. allowPrivilegeEscalation: false?
12. capabilities.drop: [ALL] + только нужные add?
13. seccomp profile: RuntimeDefault или custom?
14. AppArmor/SELinux profile?
15. no-new-privileges?

### Network
16. NetworkPolicy ingress + egress?
17. Нет hostNetwork: true?
18. Порты разрешены только нужные?

### Secrets
19. Secrets через Vault / External Secrets / Sealed Secrets?
20. Нет plain k8s Secrets в git?
21. ServiceAccount token automount: false?

### Cluster-level
22. Pod Security Admission: restricted?
23. Kyverno/Gatekeeper policies?
24. Falco/Tetragon runtime detection?
25. RBAC: least privilege?

Для каждого пункта:
- ✅ OK / ⚠️ Warning / ❌ Critical
- Конкретный fix с YAML/Dockerfile-фрагментом
- Оценка риска (low/medium/high)

В конце:
- Сводный риск-score
- Топ-5 критичных проблем
- Исправленные манифесты и Dockerfile

---

## Мои файлы

### Dockerfile
```dockerfile
<вставь>
```

### k8s manifest
```yaml
<вставь>
```

---

- [stages/05-security.md](../stages/05-security.md)
