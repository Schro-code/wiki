---
domain: azure
tags: [azure, az204]
sources: []
updated: 2026-10-30
---

# Azure Functions

## What it is

Azure's serverless compute service: your code runs only when an event fires (an HTTP request, a queue message, a timer, a blob upload), Azure scales instances automatically, and you pay only for actual executions (first million free per month).

## What problem it solves

You have a process that only needs to run when something happens — a message arrives, someone uploads a file, a timer fires. With a traditional app you pay for the server 24/7 even if it is idle 90% of the time.

**Azure Functions** solves this with the **serverless** model:
- Your code only runs when there is an event
- Azure scales instances automatically based on load
- You only pay for actual executions (first million free per month)

```
Traditional problem:
  Server up 24/7 → you pay 24/7, even with no work

With Functions:
  Event happens → Azure starts the function → processes → shuts down
  0 work → 0 cost
```

> A Function App **always needs an associated Storage Account**. It uses it internally to manage trigger state, internal queues and execution logs. Without a storage account, the Function App cannot start.

## Notes

### Triggers

What **starts** the function. Every function has **exactly one trigger** — never zero, never two.

| Trigger | When it runs |
|---|---|
| `httpTrigger` | When an HTTP request arrives |
| `queueTrigger` | When a message arrives in a Storage Queue |
| `timerTrigger` | On a cron-style schedule (`0 */5 * * * *`) |
| `blobTrigger` | When a blob is uploaded or modified |
| `eventHubTrigger` | When an event arrives in Event Hubs |
| `serviceBusTrigger` | When a message arrives in Service Bus |

### Bindings

Declarative connections to other services. Instead of writing code to connect to a queue or a table, you declare to Azure what you want to read or write and it injects it (or writes it) for you.

| Type | Direction | What it means |
|---|---|---|
| Trigger | `in` | Starts the function; Azure hands you the data |
| Input binding | `in` | Reads additional data when executing |
| Output binding | `out` | Writes data when the function finishes |

`direction` only has two values: `"in"` or `"out"`. Triggers are always `"in"`.

#### The arg_name / name connection

The `arg_name` field in the decorator (v2) or `"name"` in function.json (v1) **must match exactly** the parameter name in the code. It is what the runtime uses to know what to inject where.

```
v1 function.json          v2 decorator              function parameter
──────────────────────────────────────────────────────────────────────────
"name": "message"    ←→  arg_name="message"    ←→  message: func.QueueMessage
"name": "auditLog"   ←→  arg_name="auditLog"   ←→  auditLog: func.Out[dict]
```

If the string does not match exactly, the runtime cannot inject the value and the function fails on startup.

### func.Out — how set() works

`func.Out[T]` is a **local container**. When you call `.set(value)`, the value is stored in memory inside that container — Azure **does not write anything yet**.

The runtime writes all outputs to Azure when the function finishes successfully. If the function throws an exception before finishing, the outputs are discarded and nothing is written.

#### Why this behavior exists

Azure needs to know the function finished correctly before writing. If it wrote on every `.set()` and the function then failed halfway, you would have written partial or incorrect data to your database or queue — with no way to undo it.

#### Concrete example

```python
@app.queue_trigger(arg_name="message", queue_name="notifications",
                   connection="AzureWebJobsStorage")
@app.table_output(arg_name="auditLog", table_name="Audit",
                  connection="AzureWebJobsStorage")
def process(message: func.QueueMessage, auditLog: func.Out[dict]):

    data = message.get_json()

    # This .set() does NOT write to Table Storage yet.
    # It only stores the value in the local container.
    auditLog.set({
        "PartitionKey": "policies",
        "RowKey": data["policy_id"],
    })

    # If an exception were raised here, auditLog would never be written to Azure.
    # The function finishes successfully → Azure writes auditLog now.
```

**Typical use case:** when you need to perform several operations before confirming that everything went well. Call `.set()` at the end of the correct block; if something fails earlier, no incomplete record is left behind.

### $return — special output binding

`$return` is a reserved name that tells the runtime the output binding's value is the function's `return`, not an `Out` parameter.

**With `func.Out` (normal name):**
```python
def main(req, Response: func.Out[func.HttpResponse]):
    Response.set(func.HttpResponse("ok"))
```

**With `$return` (the function's return):**
```python
def main(req) -> func.HttpResponse:
    return func.HttpResponse("ok")  # the return IS the output binding
```

In v2, writing `-> func.HttpResponse` in the signature enables `$return` automatically.

> **Limitation:** there can be only one `$return` per function. If you need several outputs, the additional ones must use `func.Out` with a normal name.

### Internal runtime architecture

When you deploy a Function App there are **two processes** running:

```
┌────────────────────────────────────────┐
│         Azure Functions Host           │  ← Azure's runtime
│                                        │
│  - Reads host.json and function.json   │
│  - Listens for triggers (queue, HTTP…) │
│  - Manages input/output bindings       │
│  - Scales instances                    │
│                                        │
│  When an event arrives:                │
│    → calls the Language Worker         │
└────────────────┬───────────────────────┘
                 │  (internal call)
         ┌───────▼───────┐
         │ Language Worker│  ← process that runs YOUR code
         │  (Python, C#,  │
         │   JS, Java...) │
         └───────────────┘
```

The host takes care of all the Azure "plumbing" (triggers, bindings, scaling). The Language Worker is the process that runs your code — there is one native worker for each supported language (Python, C#, JS, Java, PowerShell). Communication between host and worker uses gRPC, a proprietary Azure protocol.

### Custom Handlers

#### The problem

If you want to use a language that is not natively supported (Go, Rust, Ruby...) there is no Language Worker for it. Custom Handlers solve this by changing who receives the call: instead of a worker over gRPC, the host makes an **HTTP request to your own process**.

Your process is a regular HTTP server that you write in whatever language you want. The host starts it, sends it the event as JSON, and waits for a JSON response with the outputs.

#### Configuration in host.json

```json
{
  "version": "2.0",
  "customHandler": {
    "description": {
      "defaultExecutablePath": "myapp",
      "arguments": [],
      "workingDirectory": ""
    },
    "enableForwardingHttpRequest": false
  }
}
```

- `defaultExecutablePath` — the binary Azure starts (your Go binary, your Rust binary...)
- `enableForwardingHttpRequest` — if `true`, for HTTP triggers it forwards the original request as-is instead of wrapping it in JSON. Simpler, but only works with HTTP triggers

Each function's `function.json` does not change — you define trigger and bindings as usual.

#### The payload your server receives

For any trigger that is not a direct HTTP one, the host sends:

```json
POST /FunctionName

{
  "Data": {
    "queueItem": "the queue message"
  },
  "Metadata": {
    "DequeueCount": "1",
    "InsertionTime": "2026-07-28T10:00:00Z"
  }
}
```

And your server responds:

```json
{
  "Outputs": {
    "res": { "statusCode": 200, "body": "processed" }
  },
  "ReturnValue": null,
  "Logs": ["processing message..."]
}
```

#### The 60-second limit

When Azure starts your binary, it waits for the HTTP server to be ready to accept requests. The limit is **60 seconds** — if your server is not listening before that time passes, Azure considers the startup failed and the Function App does not initialize.

> For the exam: if the question says the server starts in ≤ 60 seconds, it meets the requirement. If it said 120 seconds, it would not.

### Application Insights

To connect a Function App to Application Insights, you only need to add an app setting with the resource's connection string:

```
APPLICATIONINSIGHTS_CONNECTION_STRING = InstrumentationKey=a1b2c3d4-xxxx;IngestionEndpoint=https://westeurope-5.in.applicationinsights.azure.com/;LiveEndpoint=https://westeurope.livediagnostics.monitor.azure.com/
```

| Field | What it is |
|---|---|
| `InstrumentationKey` | GUID that identifies the App Insights resource. Exclusive to this service |
| `IngestionEndpoint` | HTTPS URL where telemetry (invocations, errors, traces) is sent |
| `LiveEndpoint` | URL for the Live Metrics stream (real-time metrics) |

With this variable present, the host sends telemetry automatically — without touching the code.

> If you want telemetry separated by environment (dev, staging, prod), each Function App must have its own connection string pointing to a different App Insights resource. `host.json` lives in the repository and is the same for all environments, so it is not the right place for this.

### AuthLevel (httpTrigger only)

Controls who can call the HTTP function.

| Level       | Who can call                                                                 |
| ----------- | ---------------------------------------------------------------------------- |
| `ANONYMOUS` | Anyone, without authentication                                               |
| `FUNCTION`  | Only whoever has that function's API key (`?code=xxx` in the URL) (DEFAULT)  |
| `ADMIN`     | Only whoever has the Function App's master key                               |

```python
# Anyone can call this function
@app.route(route="public", auth_level=func.AuthLevel.ANONYMOUS)
def public_endpoint(req: func.HttpRequest) -> func.HttpResponse: ...

# Only with ?code=<api-key> in the URL
@app.route(route="private", auth_level=func.AuthLevel.FUNCTION)
def private_endpoint(req: func.HttpRequest) -> func.HttpResponse: ...
```

### Durable Functions

#### The problem it solves

Regular Azure Functions are **stateless** — they start, do something, finish, and forget everything. They have no memory between executions.

That is a problem when you need a process with several chained steps, one that waits hours or days for something to happen, or one that launches tasks in parallel and needs to wait for all of them to finish.

**Durable Functions** adds state and coordination without you managing queues, timers or retries.

#### What it does

The orchestrator is a function that **coordinates** other functions (called activities), but does not call them directly. It calls them through the Durable Functions context, which allows Azure to put the orchestrator to sleep and resume it when needed.

```
Regular function:
  result = do_something()   ← direct call, blocks until it finishes

Orchestrator:
  result = yield context.call_activity("DoSomething", data)
           ↑
           Azure puts the orchestrator to sleep here,
           starts the activity separately,
           and when it finishes resumes the orchestrator with the result
```

With `yield` the orchestrator hands control to Azure — it is not blocked waiting, it is shut down without consuming resources. When the activity finishes, Azure resumes it.

**Activities** are regular Azure Functions: they do real work, can do I/O, can fail and be retried.

> The orchestrator must be **deterministic** — no `datetime.now()`, no `random`, no direct I/O. All of that goes in the activities. If the orchestrator were not deterministic, on resume it would produce different results and the state would be corrupted.

#### Application patterns

| Pattern | What it does | Typical case |
|---|---|---|
| **Function Chaining** | Runs functions in sequence; one's output is the next one's input | Processing pipeline where each step depends on the previous one |
| **Fan-out / Fan-in** | Launches several functions in parallel and waits for all to finish before continuing | Process N files in parallel and consolidate the result |
| **Async HTTP APIs** | Immediately returns a `202 Accepted` with a status URL; the client polls until it finishes | Long operations where the client cannot wait blocked |
| **Monitor** | Loop that periodically checks something with variable logic — can change the interval or stop depending on conditions | Wait for an external job to finish, with backoff |
| **Human Interaction** | The orchestrator pauses waiting for an external signal with a timeout; if it does not arrive in time, it escalates or cancels | Approval flows (purchases, requests, documents) |
| **Aggregator** | Accumulates data from many events over time in an entity with persistent state | Counters, state of an IoT device |

**Function Chaining**
```
F1 → F2 → F3 → final result
```

**Fan-out / Fan-in**
```
        ┌→ F1 ─┐
F0 ─────┼→ F2 ─┼─→ aggregate results
        └→ F3 ─┘
```

**Async HTTP APIs**
```
Client → POST /process → 202 + status URL
Client → GET /status/abc → {"status": "Running"}
Client → GET /status/abc → {"status": "Completed"}
```

**Monitor**
```
while condition is not met:
    wait N minutes
    check external state
finish when condition OK
```

**Human Interaction**
```
Orchestrator waits for event "Approved" or "Rejected"
  ├── If approval arrives → continues
  └── If X hours pass without response → escalates or cancels
```

**Aggregator**
```
Event A ─┐
Event B ─┼─→ Entity "Counter" { value: 42 }
Event C ─┘
```

#### How it stores state in the Storage Account

Durable Functions uses the Storage Account so it does not lose progress even when the orchestrator shuts down. It stores three things:

| What it stores | Where | What for |
|---|---|---|
| Work messages ("run this activity") | **Queues** | Communication between orchestrator and activities |
| History of what has happened | **Tables** | Knowing where it was when it resumes |
| Large data (inputs/outputs that do not fit in a table) | **Blobs** | Overflow of heavy payloads |

The history is the key. When the orchestrator resumes, Azure replays it from the beginning using that history — that is why it knows exactly where it was without having been active the whole time.

If that history grows too large (many completed orchestrations accumulated), Storage gets saturated and everything slows down. The solution is to clean it with `PurgeInstanceHistoryAsync()`.

#### Instance management — Management API

Once orchestrations are running, the **Client** gives you methods to query and control them.

##### Querying status

`getStatusAll()` — returns all instances with their current status:

```json
[
  {
    "instanceId": "order-00123",
    "runtimeStatus": "Running",
    "createdTime": "2026-07-20T08:00:00Z",
    "lastUpdatedTime": "2026-07-20T08:05:23Z"
  },
  {
    "instanceId": "order-00098",
    "runtimeStatus": "Completed",
    "createdTime": "2026-07-20T07:00:00Z",
    "lastUpdatedTime": "2026-07-20T07:02:11Z"
  },
  {
    "instanceId": "order-00071",
    "runtimeStatus": "Failed",
    "createdTime": "2026-07-20T06:00:00Z",
    "lastUpdatedTime": "2026-07-20T06:00:45Z"
  }
]
```

`runtimeStatus` can be: `Pending`, `Running`, `Completed`, `Failed`, `Terminated`, `Suspended`.

`ListInstancesAsync()` does the same but lets you filter — for example, only those in `Running` or only those created today.

##### What you can do with each instance

| Method | What it does |
|---|---|
| `getStatusAll()` | Lists all instances with their status |
| `ListInstancesAsync()` | Lists with filters (status, date, etc.) |
| `TerminateAsync(instanceId)` | Kills a running instance |
| `RaiseEventAsync(instanceId, event)` | Sends an external event (approval, signal) |
| `PurgeInstanceHistoryAsync()` | **Deletes** the history of completed instances — frees up Storage space |

> `Purge` and `Delete` remove history, they do not show it to you. They are for cleaning up, not for diagnosing. If the question asks to *check the status*, the answer is `getStatusAll` or `ListInstancesAsync`.

### AZ-204 exam keys

- A function has **exactly one trigger**; it can have zero or more input/output bindings
- **A Storage Account is always required** — it is an infrastructure requirement, not optional
- `direction: "in"` = trigger or input binding; `direction: "out"` = output binding
- `func.Out[T]` → `.set()` stores in memory; Azure writes when the function finishes successfully
- If the function throws an exception → the outputs are **not written**
- HTTP trigger → returning `func.HttpResponse` is mandatory. Queue/Timer/Blob → no return
- `"name": "$return"` → the output is the function's `return`. Only one per function
- v1: `function.json` + `__init__.py`. v2: decorators in `function_app.py`
- `AuthLevel.FUNCTION` → needs `?code=` in the URL
- `AuthLevel.ANONYMOUS` → no authentication
- **Custom handler** → unsupported language → your process is an HTTP server → the host POSTs the event to it
- A custom handler is configured in `host.json`, not in `function.json`
- The custom handler server has **60 seconds** to be ready — if it takes longer, the Function App does not start

## Relations

- [[azure-application-insights]] — telemetry for a Function App is enabled with the `APPLICATIONINSIGHTS_CONNECTION_STRING` app setting
- [[azure-event-hubs]] — `eventHubTrigger` fires a function when an event arrives in Event Hubs
- [[azure-service-bus]] — `serviceBusTrigger` fires a function when a message arrives in Service Bus
- [[azure-event-grid]] — a function is a typical Event Grid handler
- [[azure-logic-apps]] — low-code alternative that uses connectors instead of code; a Logic App can call a Function as an action
- [[app-service-deployment-slots]] — the Consumption plan (Azure Functions) does not support deployment slots

## Tags

#azure #az204
