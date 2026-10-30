---
domain: azure
tags: [azure, az204]
sources: []
updated: 2026-10-30
---

# Azure Container Services

## What it is

Azure offers four different ways to run containers: Azure Container Instances (ACI), App Service (Web App for Containers), Container Apps and Azure Kubernetes Service (AKS). The key difference between them is how much control you want versus how much Azure manages for you.

```
Simplicity  →  ACI → App Service → Container Apps → AKS  →  Control
```

## What problem it solves

Running a container used to mean provisioning infrastructure yourself. Each of the four services removes a different amount of that burden, so you can pick the lowest level of operational work that still covers your needs.

## Notes

### The four services

#### ACI — Azure Container Instances

**Purpose:** Run a single container immediately, with no infrastructure.

> Before ACI you had to set up a VM, install Docker and configure it just to run one container. ACI removes that step: you provide an image and within seconds the container is running.

**When to use it:** One-off tasks, jobs, quick tests, one-shot processing.

---

#### App Service (Web App for Containers)

**Purpose:** Host a web application or API in production without managing servers.

> Without App Service you would have to manage the OS, the runtime, SSL certificates, logs and restarts. App Service abstracts all of that.

**Two modes:**

| Mode | What you do |
|------|-----------|
| Without a container | You upload code — Azure provides the runtime (Python, Node, Java...) |
| With a container | You upload your Docker image — this is *Web App for Containers* |

*Web App for Containers* is not a separate service: it is App Service in container mode.

**When to use it:** An API or website in production, without wanting to manage infrastructure.

---

#### Container Apps

**Purpose:** Run several microservices with autoscaling without managing Kubernetes.

> When you have microservices that need to scale independently and talk to each other, you used to need Kubernetes. Container Apps gives you those capabilities without having to learn or operate K8s.

**When to use it:** Several services that talk to each other, need to scale to zero.

---

#### AKS — Azure Kubernetes Service

**Purpose:** Managed Kubernetes with full control over the container infrastructure.

> Kubernetes is the industry standard for orchestrating containers at scale. AKS removes the burden of building and maintaining the cluster — Azure manages the control plane, you manage the nodes and workloads.

**When to use it:** You already use Kubernetes, you need full control, you have a dedicated DevOps team.

---

### Decision table

| Question | Service |
|----------|----------|
| A one-off container / job? | **ACI** |
| An API or website in production? | **App Service** |
| Several microservices? | **Container Apps** |
| Already using Kubernetes? | **AKS** |

## Relations

- [[app-service-tiers-and-scaling]] — scaling options for the App Service option
- [[app-service-deployment-slots]] — release mechanism for App Service
- [[dns-records-custom-domains]] — attaching a custom domain to an App Service app

## Tags

#azure #az204
