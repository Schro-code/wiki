---
domain: devops
tags: [devops]
sources: []
updated: 2026-10-16
---

# Deployment Strategies

## What it is

A deployment strategy defines how a new version of an application replaces the old one on live infrastructure. The choice controls the trade-off between simplicity, risk, cost, and how long both versions run simultaneously.

## What problem it solves

You have users actively using version 1.0. You need to install version 1.1. If you just stop 1.0 and start 1.1 there will be downtime, and if 1.1 has a bug real users are immediately impacted. Deployment strategies reduce downtime and blast radius.

## Notes

### The five main strategies

#### 1. Recreate (a.k.a. stop-and-start)

```
v1 ──────────── ✗ STOPPED
                   v2 ──────────── STARTED
                   ↑ downtime
```

- Stop all instances of v1, start all instances of v2
- **Downtime:** yes, between stop and start
- **Both versions live simultaneously:** no
- **Rollback:** stop v2, start v1 (more downtime)
- **Use when:** dev/staging environments, simplicity > availability, stateful apps that can't run in parallel

---

#### 2. Rolling update

```
v1 v1 v1 v1
v2 v1 v1 v1  ← replace 1 instance
v2 v2 v1 v1  ← replace 1 more
v2 v2 v2 v1
v2 v2 v2 v2
```

- Replace instances one at a time (or in batches)
- **Downtime:** no (requests route to remaining v1 instances during rollout)
- **Both versions live simultaneously:** yes, during the rollout window
- **Rollback:** roll forward in the other direction (slow)
- **Default in Kubernetes**
- **Risk:** if v2 has a DB migration that's incompatible with v1, the period where both run can cause errors

---

#### 3. Blue-Green

```
Load balancer
   ├── Blue environment (v1) ← traffic 100%
   └── Green environment (v2) ← traffic 0% (warm up here)

After validation:
   ├── Blue environment (v1) ← traffic 0% (kept for rollback)
   └── Green environment (v2) ← traffic 100%
```

- Run two identical environments; switch all traffic at the load balancer
- **Downtime:** no (traffic switch is near-instant)
- **Both versions live simultaneously:** yes (briefly, or until rollback window closes)
- **Rollback:** flip traffic back to Blue — instant
- **Cost:** need 2x infrastructure during the switch window
- **Use when:** need instant rollback, large or risky releases, DB migrations done separately

---

#### 4. Canary

```
Load balancer
   ├── v1 ← 95% traffic
   └── v2 ← 5% traffic (canary)

Monitor metrics. If healthy:
   ├── v1 ← 80% traffic
   └── v2 ← 20% traffic

Continue until:
   └── v2 ← 100% traffic
```

- Route a small fraction of real traffic to the new version; increase the % over time as confidence grows
- **Downtime:** no
- **Both versions live simultaneously:** yes, for the entire rollout duration
- **Rollback:** reduce v2 traffic to 0%, no one else noticed
- **Named after:** canary in a coal mine — real users in the canary group detect problems before everyone is affected
- **Use when:** need gradual exposure, risk-sensitive features, A/B testing, large user bases
- **Risk:** real users hit the new version, even if it's a small %; must have good observability to detect problems

---

#### 5. Feature flags (decoupled deployment)

Not strictly a deployment strategy, but a technique used alongside any of the above to decouple **deployment** from **release**:

- Deploy v2 to all servers (code is live)
- The new feature is gated by a runtime flag: `if feature_flags.is_enabled("new-checkout")`
- Release means: flip the flag on (in a dashboard, no new deployment needed)
- Target specific users: "enable for 10% of users", "enable for beta users", "enable for employee accounts only"

**Benefits:**
- Instant rollback: just flip the flag off
- Trunk-based development: unfinished features are in prod but invisible
- A/B testing: compare v1 (flag off) vs v2 (flag on) on real traffic

**Trade-offs:**
- Flags accumulate in code and become tech debt — need a cleanup process
- Adds conditional complexity to the code
- Requires flag management infrastructure (LaunchDarkly, Unleash, custom Redis config)

---

### Comparison table

| Strategy | Downtime | Blast radius | Rollback speed | Cost |
|---|---|---|---|---|
| Recreate | Yes | 100% | Slow (more downtime) | Lowest |
| Rolling | No | Partial (% of instances) | Slow | Low |
| Blue-Green | No | 0% if Blue is kept | Instant | 2x infra |
| Canary | No | Small (% of traffic) | Fast (reduce %) | Low overhead |
| Feature flags | No | Configurable | Instant (flip flag) | Flag infra |

### Database migrations — the hard part

Every strategy has one common problem: **what about the database?**

If v2 renames a column that v1 reads, the rolling period where both run will cause errors. The standard solution is the **expand-contract** (or parallel change) pattern:

1. **Expand** — v2 adds the new column and writes to both old and new. v1 keeps working.
2. **Migrate** — backfill data into the new column.
3. **Contract** — once all instances are on v2, remove the old column.

Never do a breaking schema change at the same time as a code deploy.

## Relations

- [[environments]] — each environment uses a deployment strategy
- [[ci-cd-pipeline]] — the deploy step at the end of the pipeline uses one of these strategies
- [[devops-overview]] — reducing blast radius is a core Flow and Feedback concern

## Tags

#devops
