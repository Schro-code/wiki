---
domain: kubernetes
tags: [kubernetes]
sources: []
updated: 2026-08-25
---

# Minikube — Local Kubernetes Cluster

## What it is

Minikube runs a single-node Kubernetes cluster on your local machine for development and learning.

## What problem it solves

Setting up a real Kubernetes cluster requires multiple machines and significant infrastructure. Minikube gives you a fully functional single-node cluster locally so you can develop, test manifests, and learn Kubernetes without cloud costs or infrastructure complexity.

## Notes

### Start the cluster

```bash
minikube start
```

### Point Docker to Minikube's daemon

Run this in every new terminal before building an image. Without it, Kubernetes can't see locally built images.

```bash
eval $(minikube docker-env)
```

Then build normally:

```bash
docker build -t hello-world-fastapi:latest ./app
```

### Get the URL of a service

```bash
minikube service <service-name>
```

Or with a real EXTERNAL-IP (keep terminal 1 open):

```bash
# Terminal 1:
minikube tunnel

# Terminal 2:
kubectl get service <service-name>
curl http://<EXTERNAL-IP>
```

### Known issue — Docker credential error

**Symptom:**
```
error getting credentials - err: exec: "docker-credential-desktop": executable file not found in $PATH
```

**Fix:**

```bash
python3 -c "
import json
with open('/Users/LDAZZH/.docker/config.json') as f:
    c = json.load(f)
c.pop('credsStore', None)
with open('/Users/LDAZZH/.docker/config.json', 'w') as f:
    json.dump(c, f, indent=2)
"
```

## Relations

- [[kubectl-commands]] — commands to deploy and inspect resources
- [[kustomize]] — multi-environment manifests deployed with `kubectl apply -k`

## Tags

#kubernetes
