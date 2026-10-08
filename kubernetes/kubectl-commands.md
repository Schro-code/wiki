---
domain: kubernetes
tags: [kubernetes]
sources: []
updated: 2026-08-25
---

# kubectl — Commands

## What it is

`kubectl` is the CLI to interact with a Kubernetes cluster — deploy, inspect, and manage resources.

## Notes

### Apply and delete manifests

```bash
kubectl apply -f k8s/base/          # deploy all manifests in a folder
kubectl delete -f k8s/base/         # delete all resources in a folder

kubectl apply -k k8s/overlays/dev   # deploy via Kustomize overlay
kubectl delete -k k8s/overlays/dev  # delete via Kustomize overlay

kubectl kustomize k8s/overlays/dev  # preview the merged YAML without applying
```

### Check state

```bash
kubectl get pods          # list pods and their status
kubectl get pods -w       # watch pods in real time
kubectl get services      # list services and their external IPs
```

### Restart pods

```bash
kubectl rollout restart deployment/<name>   # force pods to restart (picks up new image)
```

## Relations

- [[minikube]] — how to start the cluster and build images locally
- [[kustomize]] — base + overlay pattern used with `kubectl apply -k`

## Tags

#kubernetes
