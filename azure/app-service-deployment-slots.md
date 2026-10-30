---
domain: azure
tags: [azure, az204]
sources: []
updated: 2026-10-30
---

# App Service Deployment Slots

## What it is

A deployment slot is an **independent instance of the app** with its own URL, its own deployed code and its own settings:

```
myapp.azurewebsites.net              ← Production slot
myapp-testing.azurewebsites.net      ← Testing slot
myapp-development.azurewebsites.net  ← Development slot
```

Each slot runs separately. You upload code to the Development slot, and that slot has that code. The others keep theirs.

> Slots require the **Standard** tier or higher. The Consumption plan (Azure Functions) does not support slots.

## What problem it solves

Deploying straight to production risks downtime and cold starts. Slots let you deploy and validate a new version in a production-like instance, warm it up, and then swap it into production atomically with no downtime.

## Notes

### Swap — how the exchange works

The swap exchanges traffic between two slots (normally staging → production). It happens in **two phases**:

**Phase 1 — warm-up:**
1. Azure applies the settings of the *target* slot (production) to the instances of the *source* slot (staging)
2. It restarts them with that configuration
3. It waits for each instance to respond HTTP 200 — only then does it consider the slot ready

**Phase 2 — the actual swap:**
Once all instances are warm, Azure atomically swaps the traffic routing, with no downtime.

> The warm-up uses the *target's* settings, not the source's. That way, by the time the swap arrives, the instances are already ready with the correct configuration.

### Swapped configuration vs sticky configuration

The most important distinction for the exam:

| Setting type | Behavior on swap |
|-----------------|----------------------|
| **Normal** (non-sticky) | Travels with the code — swapped between slots |
| **Slot setting** (sticky) | Stays in the slot — **NOT** swapped |

**Key example:** the production database connection string is marked as sticky → even if you swap, the production slot always points at the production DB, regardless of the code it holds.

### Custom warm-up with `web.config`

By default Azure only waits for HTTP 200 on `/`. But if your app needs to run scripts, pre-load cache or verify connections before receiving real traffic, you need to control when it is "ready".

**Solution:** add `applicationInitialization` in `web.config` (the IIS configuration file — IIS is the Windows web server under App Service):

```xml
<applicationInitialization>
  <add initializationPage="/statuscheck" />
</applicationInitialization>
```

This tells IIS: *"before declaring the slot ready, call `/statuscheck` and wait for an HTTP 200"*.

**You implement the `/statuscheck` endpoint** in the app's code — the app decides when it is ready. There you run initialization scripts, verify resources, etc. Only when everything is correct does it return 200 and the swap continues.

> Exam trick: IIS calls the endpoint locally; it is not Azure calling from outside.

### Auto-swap

You can configure it so that when you deploy to a slot (e.g. staging), the swap to production happens **automatically** with no manual intervention. Useful in CI/CD pipelines.

It is configured per slot, not per app. In the typical exam question: *"auto swap enabled on the Production slot"* means that when staging receives a deployment, the swap to production fires on its own.

### Swap with preview

Lets you pause the process at Phase 1 — the settings are already applied but traffic has not yet been swapped. It is used to manually validate that everything works before confirming the swap.

## Relations

- [[app-service-tiers-and-scaling]] — slots require Standard tier or higher
- [[azure-container-services]] — App Service in context with the other container/hosting options
- [[azure-app-configuration]] — centralized settings and feature flags as an alternative to per-slot settings

## Tags

#azure #az204
