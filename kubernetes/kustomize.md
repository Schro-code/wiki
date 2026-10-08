---
domain: kubernetes
tags: [kubernetes]
sources: []
updated: 2026-08-25
---

# Kustomize — Multi-Environment Manifest Management

## What it is

Kustomize is a tool built into `kubectl` (since v1.14) that lets you customize Kubernetes manifests per environment without duplicating YAML files.

## What problem it solves

Without Kustomize, managing different configurations for dev and prod means either duplicating all YAML files or editing them manually before each deploy. Kustomize introduces a **base + overlay** pattern: write the YAML once, describe only the differences per environment.

## Notes

### File structure

```
k8s/
├── base/                        # environment-agnostic manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml       # lists the resources in base
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml   # points to base + applies patch.yaml
    │   └── patch.yaml           # only the fields that differ in dev
    └── prod/
        ├── kustomization.yaml
        └── patch.yaml
```

### base/kustomization.yaml

Just an index of the files that make up the project:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

### overlays/dev/kustomization.yaml

Points to base and declares which patch to apply:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - path: patch.yaml
    target:
      kind: Deployment
      name: hello-world
```

### overlays/dev/patch.yaml

A strategic merge patch — only the fields to override, not the whole file:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: hello-world
          env:
            - name: ENV
              value: "dev"
```

Kustomize merges this on top of the base Deployment. Every field not listed here stays unchanged from base.

### Commands

```bash
# Deploy a specific overlay
kubectl apply -k k8s/overlays/dev
kubectl apply -k k8s/overlays/prod

# Preview the final merged YAML without applying
kubectl kustomize k8s/overlays/dev

# Delete resources created by an overlay
kubectl delete -k k8s/overlays/dev
```

### Mental model

```
base deployment  +  patch  =  final YAML applied by kubectl
(replicas: 5)      (replicas: 2)   → replicas: 2
(ENV: local)       (ENV: dev)      → ENV: dev
(everything else unchanged)
```

## Relations

- [[kubectl-commands]] — `kubectl apply -k` and `kubectl kustomize` commands
- [[minikube]] — how to run the cluster where Kustomize manifests are deployed

## Tags

#kubernetes
