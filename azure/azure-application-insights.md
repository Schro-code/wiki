---
domain: azure
tags: [azure, az204]
sources: []
updated: 2026-10-30
---

# Azure Application Insights

## What it is

A monitoring service for **applications** (not infrastructure) within Azure Monitor. It detects performance issues, failures and user behavior.

## What problem it solves

| Without App Insights | With App Insights |
|---|---|
| You don't know what fails inside your app | Traces of exceptions and dependencies |
| You don't know what users do | Usage Analytics with custom events |
| You don't know if the app is slow | Response times and latencies |

**Key difference with Azure Monitor:** Monitor covers infrastructure (VMs, Event Hubs, platform metrics). App Insights covers what happens *inside* the code.

## Notes

### The 5 telemetry types

| Type | What it is | Example |
|---|---|---|
| **Request** | Incoming HTTP request to your app | GET /api/products → 200, 45ms |
| **Dependency** | Call your app makes to something external | SQL query, call to another API, Blob Storage |
| **Exception** | Unhandled or handled error | NullReferenceException at line 42 |
| **Trace** | Structured text log | "Starting payment process" |
| **Custom Event** | Business event that you define | "ProductAddedToCart" |

**Request** and **Dependency** give the performance map. **Exception** gives the failures. **Custom Event** gives the business behavior.

---

### How it is used

**Option 1 — SDK (manual instrumentation):**
```csharp
var telemetry = new TelemetryClient();
telemetry.TrackEvent("ProductViewed");
telemetry.TrackException(ex);
telemetry.TrackTrace("log message");
```

**Option 2 — Automatic instrumentation (no code):**
In App Service you can enable App Insights from the portal. The agent injects telemetry without modifying the code.

**Connection String** (not Instrumentation Key — it is deprecated):
```
InstrumentationKey=xxx;IngestionEndpoint=https://...
```

---

### Cost control

App Insights charges per **GB of telemetry ingested**. Three mechanisms to limit it:

| Mechanism | Where it is configured | How it works | Guarantees a cap? |
|---|---|---|---|
| **Daily Cap** | Azure portal | Stops ingestion when it reaches the daily GB limit | Yes — hard stop |
| **Adaptive Sampling** | SDK (code) | The SDK automatically reduces the % of events sent depending on volume | No — reduces, doesn't cap |
| **Ingestion Sampling** | Azure portal | Azure discards a % of data after receiving it | No — reduces, doesn't cap |

---

### How Application Insights works

Everything goes to a **Log Analytics Workspace**: a log database managed by Azure. Application Insights is basically a layer on top of that workspace with predefined tables (`requests`, `exceptions`, `traces`...).

#### Analysis tools (what you see in the portal)

They are different views over the same data — they are not separate things, they are different lenses:

| Tool | What it captures / queries | Example |
|---|---|---|
| **Live Metrics** | Live stream — not persisted in the workspace, visualization only | You just deployed. You watch requests/sec, failure rate and server CPU live |
| **Application Map** | Data from `requests` and `dependencies` — draws the dependency graph automatically | Your app calls SQL and an external API. The SQL node appears in red → that's where the problem is |
| **Failures** | `requests` with `success=false` and the `exceptions` table, grouped by type and frequency | `NullReferenceException` happened 47 times today. You click and reach the exact stack trace of each instance |
| **Performance** | Time distribution of `requests` and `dependencies` | The 95th percentile of `/api/checkout` is 4s. Drill-down: the SQL call inside takes 3.8s |
| **Availability** | `availabilityResults` table — results of the ping tests that you configure | Test from 5 regions: West Europe fails 12%. The rest are fine |
| **Logs** | Direct access to all tables via KQL — the most flexible | Query: failed requests in the last hour grouped by endpoint |
| **Metrics** | Pre-aggregated metrics — faster and cheaper than raw KQL | Chart with `requests/duration` and `exceptions/count` together to correlate spikes |
| **Users / Sessions / Events** (Usage Analytics) | `customEvents` and `pageViews` — requires `TrackEvent()` or the JS SDK | 1,000 viewed the product, 200 added to cart, 30 paid → the drop is at checkout |

#### Specialized tools (enabled separately)

**Application Insights Profiler** — for performance
> Captures detailed CPU traces when a request takes too long. It tells you exactly which line of code is consuming time.

**Snapshot Debugger** — for errors
> Captures a snapshot of the memory state at the exact moment an exception occurs. It lets you see the values of the variables when something failed, without needing to reproduce it.

#### The full picture

```
Your app
  │
  ├── SDK / Auto-instrumentation
  │        │
  │        ▼
  │   Application Insights Resource
  │        │
  │        ▼
  │   Log Analytics Workspace  ←── KQL queries (Logs)
  │        │
  │        ├── Live Metrics        (real time)
  │        ├── Application Map     (visual)
  │        ├── Failures            (errors)
  │        ├── Performance         (slowness)
  │        ├── Availability        (ping tests)
  │        └── Metrics / Workbooks (dashboards)
  │
  ├── Profiler          (enabled separately → CPU analysis)
  └── Snapshot Debugger (enabled separately → exception analysis)
```

---

### Application Insights Profiler

**The problem it solves:** you have an app in production. You see in Performance that a request takes 3 seconds. But you don't know why — it could be a SQL query, an HTTP call, a badly written loop. Profiler answers exactly that: which line of code is consuming the time.

#### How it works

Profiler acts as a **CPU sampler**. Every so often it hooks into your app's process and captures the **call stack** at that instant. It does this thousands of times per second.

```
Request arrives → Profiler records call stack snapshots → Request ends
                         ↓
              Uploads the trace to Application Insights
```

The result is a **flame graph**: each block is a method, the width indicates time consumed. The **hot path** is the most expensive route — it is the first thing you look at to optimize.

#### Storage

Profiler traces **do not go to the Log Analytics Workspace** — they are stored in an **Azure Blob Storage** linked to the Application Insights resource.

#### On-demand vs real-time

The **on-demand** mode does capture at the moment: you activate it, it records for ~2 minutes while real requests arrive, and you see the result almost immediately.

But it is **not real-time in the same sense as Live Metrics**, which is a continuous stream with ~1s latency where you see metrics flowing live. The difference:

| | Profiler on-demand | Live Metrics |
|---|---|---|
| What it does | Captures a ~2 min block and then you analyze it | Continuous data stream |
| Result | A trace that you examine afterwards | Charts updating live |
| What for | "I want to record now to analyze" | "I want to see what is happening right now" |

---

### Usage Analytics

The 5 telemetry types answer **"what fails and what is slow?"**. Usage Analytics answers **"what do users do?"**. You can have an app without errors that nobody uses well.

It requires custom events — without `TrackEvent()` there is no data of your own.

| Tool | What it shows | Example |
|---|---|---|
| **Users** | Unique people (by cookie or ID) | You have 200 users/day. On Monday you see 20 — something happened that day |
| **Sessions** | Grouped visits; a user can have several | 500 users and 501 sessions → nobody comes back. 500 users and 2,000 sessions → they use it several times a day |
| **Events** | Actions that you define with `TrackEvent()` | 1,000 users view products but only 50 add them to the cart → the button doesn't look right |
| **Funnels** | % that completes each step of a flow | 80% add to cart, 10% pay → the problem is in checkout, not in the catalog |
| **Retention** | How many users return after X days | You launch a feature: after 7 days 60% came back vs the previous 20% → the feature is engaging |
| **User Flows** | Real paths that users follow | You thought they went Home → Catalog → Product. They go Home → Search → Product. The catalog is invisible |
| **Impact** | How performance affects behavior | The checkout page takes 5s → 70% abandon. If it drops to 1s → abandonment falls to 20% |

---

### Snapshot Debugger

**The problem it solves:** you have an exception in production. The stack trace tells you where it failed, but not why — you don't know what values the variables had at that moment. Reproducing it locally is impossible or takes hours.

Snapshot Debugger captures the **exact memory state** at the moment the exception occurs — local variables, method parameters, call stack — without needing to reproduce it.

#### How it works

```
Exception occurs in production
        ↓
Snapshot Debugger detects it
        ↓
Captures the memory state at that instant
        ↓
Uploads the snapshot to Azure Blob Storage
        ↓
You see it in Application Insights → Failures → Open Debug Snapshot
```

**Default threshold:** it does not capture on the first occurrence. It waits for the same exception to occur **5 times** before capturing. Configurable with `ThresholdForSnapshotting`. It avoids saturating storage with massive errors.

**Retention:** snapshots are kept for **15 days**.

**Storage:** same as Profiler — in **Azure Blob Storage**, not in the Log Analytics Workspace.

---

### What works and what doesn't on the Consumption plan

Profiler and Snapshot Debugger need a **persistent agent hooked into the process**. On the Consumption plan instances are ephemeral and scale to zero — there is no stable process to hook into.

Standard telemetry does work because it only needs to **send data over HTTP** to the Application Insights endpoint — it requires no persistence.

| | Consumption | Premium |
|---|---|---|
| Requests, exceptions, traces, events | Yes | Yes |
| Live Metrics, Failures, Performance | Yes | Yes |
| Logs / KQL, Alerts, Availability | Yes | Yes |
| **Profiler** | **No** | Yes |
| **Snapshot Debugger** | **No** | Yes |

---

### AZ-204 exam keys

- **Budget cap vs cost reduction:** if the question says *"do not exceed a budget"* → **Daily Cap**. If it says *"reduce costs"* → sampling (adaptive if you have access to the SDK, ingestion if you can't touch the code).
- **Consumption plan rule:** if it requires a persistent agent → it does not work on Consumption. If it only sends data → it does work.

## Relations

- [[azure-functions]] — Functions on the Consumption plan support standard telemetry but not Profiler or Snapshot Debugger
- [[azure-blob-storage]] — where Profiler traces and debug snapshots are stored (not Log Analytics)
- [[azure-event-hubs]] — platform metrics for services like Event Hubs belong to Azure Monitor, not App Insights

## Tags

#azure #az204
