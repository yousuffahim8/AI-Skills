# Kubernetes Investigation Commands (EKS or AKS)

Get credentials first (`aws eks update-kubeconfig --name <cluster>` or `az aks get-credentials -n <cluster> -g <rg>`), then confirm you are on the right cluster:

```bash
kubectl config current-context
```

All commands below are read-only.

```bash
# Pods that are not healthy
kubectl get pods -A | grep -vE 'Running|Completed'

# Restart counts (crash loops show up here first)
kubectl get pods -n <ns> --sort-by='.status.containerStatuses[0].restartCount'

# Why a pod is failing: events, probe failures, OOMKilled, image pull errors
kubectl describe pod <pod> -n <ns>

# Logs from the crashed container, not the fresh one
kubectl logs <pod> -n <ns> --previous --tail=200

# Recent cluster events, newest last
kubectl get events -n <ns> --sort-by=.lastTimestamp | tail -30

# Rollout state and history (correlate with deploys)
kubectl rollout status deployment/<name> -n <ns>
kubectl rollout history deployment/<name> -n <ns>

# Node pressure and capacity
kubectl get nodes
kubectl top nodes
kubectl top pods -n <ns> --sort-by=memory

# Service has no endpoints = selector mismatch or no ready pods
kubectl get endpoints <service> -n <ns>
```

| Signal | Usual cause |
|---|---|
| `CrashLoopBackOff` | App exits on start: bad config, missing secret, failed dependency |
| `OOMKilled` (exit 137) | Memory limit too low or a leak |
| `ImagePullBackOff` | Wrong tag, missing registry permission |
| `Pending` | No node fits: CPU/memory requests, taints, PVC binding |
| Readiness probe failing | App up but a dependency is down, or the probe path is wrong |
