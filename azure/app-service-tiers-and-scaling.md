---
domain: azure
tags: [azure, az204]
sources: []
updated: 2026-10-30
---

# App Service Tiers and Scaling

## What it is

Which App Service tiers support automatic scaling, and which CLI commands perform automatic (metric-based) versus manual scaling across Azure resources.

## What problem it solves

When traffic is high you need to add capacity without manual intervention. Choosing the wrong tier (one that doesn't support autoscale) or the wrong command (manual instead of rule-based) leaves the app unable to react to load.

## Notes

### Automatic scaling by tier

| Tier | Automatic scaling |
|------|-------------------|
| **Shared** | No — does not support autoscaling when traffic is high |
| **Standard** | Yes — it is the cheapest option that includes autoscaling |

> If you need to scale automatically, the minimum tier is **Standard**.

### Scaling CLI commands

| Type | Command | Usage example |
|------|---------|----------------|
| **Automatic** (by metrics) | `az monitor autoscale rule create` | Scale when CPU > 85% |
| **Manual** (App Service) | `az appservice plan update` | Set the number of instances |
| **Manual** (AKS) | `az aks nodepool scale` | Change the cluster's nodes |
| **Manual** (VM Scale Sets) | `az vmss scale` | Change the number of VM instances |

> Metric-based autoscale **always** goes through `az monitor autoscale`, regardless of the resource you are scaling. Manual scaling uses each service's own namespace.

## Relations

- [[app-service-deployment-slots]] — slots also require Standard tier or higher
- [[app-service-session-management]] — what to do about session state once you scale out
- [[azure-container-services]] — AKS and App Service as scalable hosting options

## Tags

#azure #az204
