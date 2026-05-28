# ☸️ kubectl Cheatsheet

Самые нужные команды kubectl.

---

## Контекст / kubeconfig

```bash
kubectl config get-contexts
kubectl config current-context
kubectl config use-context my-cluster
kubectl config set-context --current --namespace=prod

# kubectx + kubens
kubectx prod-eu-west-1
kubens prod
```

---

## Просмотр

```bash
kubectl get pods
kubectl get pods -A                       # все namespaces
kubectl get pods -o wide                  # с IP и нодами
kubectl get pods -l app=myapp             # по label
kubectl get pods --watch                  # в реальном времени
kubectl get pods --sort-by=.metadata.creationTimestamp

kubectl get all -n mynamespace
kubectl get deploy,svc,ing -n prod

kubectl describe pod <name>
kubectl describe node <name>
kubectl explain deployment.spec.replicas    # документация
```

---

## Логи

```bash
kubectl logs <pod>
kubectl logs <pod> -f --tail=100
kubectl logs <pod> --previous              # предыдущего контейнера (важно при crashes!)
kubectl logs -l app=myapp --tail=50
kubectl logs <pod> -c <container>          # конкретный контейнер в multi-container Pod

# stern — multi-pod tail
stern -l app=myapp
stern --since=10m "myapp-.*"
```

---

## Exec / port-forward / copy

```bash
kubectl exec -it <pod> -- bash
kubectl exec <pod> -- ls /app

kubectl port-forward svc/myapp 8080:80
kubectl port-forward pod/<pod> 5432:5432

kubectl cp <pod>:/app/log.txt ./log.txt
kubectl cp ./file.txt <pod>:/tmp/

# debug (для distroless-Podов)
kubectl debug -it <pod> --image=nicolaka/netshoot --target=<container>
```

---

## Apply / create / delete

```bash
kubectl apply -f manifest.yaml
kubectl apply -k overlays/prod/            # kustomize
kubectl delete -f manifest.yaml
kubectl delete pod <name>
kubectl delete pod -l app=myapp

# Быстрый контейнер
kubectl run debug --rm -it --image=nicolaka/netshoot -- bash
kubectl run nginx --image=nginx --port=80
```

---

## Scale / rollout

```bash
kubectl scale deploy/myapp --replicas=5
kubectl rollout status deploy/myapp
kubectl rollout history deploy/myapp
kubectl rollout undo deploy/myapp
kubectl rollout undo deploy/myapp --to-revision=2
kubectl rollout restart deploy/myapp        # перезапуск (без изменений образа)

kubectl set image deploy/myapp app=myapp:v2.0
```

---

## Top / metrics

```bash
kubectl top nodes
kubectl top pods
kubectl top pods --containers              # по контейнерам внутри Pod
kubectl top pods --sort-by=memory
```

---

## ConfigMap / Secret

```bash
kubectl create configmap my-cm --from-file=config.yaml
kubectl create configmap my-cm --from-literal=KEY=value
kubectl get cm my-cm -o yaml

kubectl create secret generic my-secret --from-literal=DB_PASS=secret
kubectl get secret my-secret -o jsonpath='{.data.DB_PASS}' | base64 -d
```

---

## Events / troubleshooting

```bash
kubectl get events --sort-by=.lastTimestamp
kubectl get events --field-selector type=Warning

# Почему Pod в Pending?
kubectl describe pod <pod>

# CrashLoopBackOff
kubectl logs <pod> --previous
kubectl describe pod <pod>                 # смотри Last State + Reason

# OOMKilled
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'

# ImagePullBackOff
kubectl describe pod <pod>                 # смотри Events → "Failed to pull image"
```

---

## Output / jsonpath

```bash
kubectl get pod <name> -o yaml
kubectl get pod <name> -o json
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase

# Быстро: IP всех Podов
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'
```

---

## Patch / edit

```bash
kubectl edit deploy/myapp                  # открывает $EDITOR
kubectl patch deploy/myapp -p '{"spec":{"replicas":5}}'
kubectl patch deploy/myapp --type=json -p='[{"op":"replace","path":"/spec/replicas","value":5}]'
kubectl label pod <name> env=prod
kubectl annotate pod <name> description="test"
```

---

## Drain / cordon

```bash
kubectl cordon <node>                      # перестать распланировывать новые Podы
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>
```

---

## Polezno

```bash
# Алиасы
alias k=kubectl
alias kgp="kubectl get pods"
alias kgs="kubectl get svc"

# Completion
source <(kubectl completion bash)

# Krew — менеджер плагинов
kubectl krew install ctx ns neat tree view-secret

# Diff перед apply
kubectl diff -f manifest.yaml

# Dry-run
kubectl apply -f manifest.yaml --dry-run=server
```

---

[⬅ README](../README.md)
