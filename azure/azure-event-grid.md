---
domain: azure
tags: [azure, az204]
sources: []
updated: 2026-10-30
---

# Azure Event Grid

## What it is

An event-routing service: publishers emit events to topics, and subscriptions deliver them (push) to handlers such as Azure Functions, Logic Apps or webhooks, filtered by rules you define.

## What problem it solves

You have services that need to react when something happens in Azure (a blob uploaded, a VM deleted, a message in a queue...). Without Event Grid, each service would have to poll constantly to detect changes.

```
Without Event Grid:
  App → asks every X seconds "did anything happen?" → Azure resource
        (inefficient, delayed, costly)

With Event Grid:
  Azure resource → emits event → Event Grid → delivers it to the subscriber
                                               (immediate reaction, no polling)
```

## Notes

### Key concepts

| Concept | What it is |
|---------|--------|
| **Event** | Notification that something happened ("a blob was uploaded", "a VM was created") |
| **Publisher** | The resource that generates the event (Blob Storage, Resource Groups, your apps) |
| **Topic** | Channel where events are published |
| **Subscription** | Rule that defines which events reach which destination |
| **Handler** | The destination that processes the event (Azure Function, Logic App, webhook) |

### Event flow

```
┌──────────────────────────────────────────────────────┐
│  1. Something happens in Azure (e.g. blob uploaded   │
│     to Storage)                                      │
└─────────────────────────┬────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────┐
│  2. Event Grid receives the event and looks up       │
│     the matching subscriptions                       │
│     (filters by type, prefix, suffix...)             │
└─────────────────────────┬────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────┐
│  3. Delivers the event to the handler:               │
│     → Azure Function (processes the image)           │
│     → Logic App (sends an email)                     │
│     → Webhook (calls your API)                       │
└──────────────────────────────────────────────────────┘
```

### Topics and Subscriptions

| Concept | What it is |
|---------|--------|
| **Topic** | Channel where the publisher dumps all its events — it does not filter or deliver, it only accumulates |
| **Subscription** | Rule that defines which events from the topic are delivered and to which handler |

You can have multiple subscriptions on the same topic, and multiple topics in the same account:

| Topic type | Who creates it | When to use it |
|--------------|--------------|--------------|
| **System topic** | Azure, automatically when you enable Event Grid on a resource | When you react to events from Azure services (Blob Storage, Resource Groups, Container Registry…). They require no manual configuration. |
| **Custom topic** | You, via portal, CLI or ARM | When your own application needs to emit events (`OrderPlaced`, `PaymentFailed`). You publish via HTTP POST with your key. |
| **Domain topic** | Inside an Event Grid Domain | Multi-tenant scenarios: one topic per customer inside a shared domain. Useful in SaaS where each tenant has its own channel without paying for an independent topic. |

**Example — banking app:**

```
Azure AD → topic "auth-events"
               │
               └── Subscription [filter: sign-out] → EventGridController
                                                     (invalidates the session instantly)
```

### The Event object

Everything that travels through Event Grid is a JSON array of events. Each event has this structure:

```json
[{
  "id": "a3c5e1d2-4f89-4b2c-9e71-abc123456789",
  "eventType": "Contoso.OrderPlaced",
  "subject": "/orders/12345",
  "eventTime": "2026-07-28T10:00:00Z",
  "data": {
    "orderId": "12345",
    "amount": 99.99,
    "currency": "EUR"
  },
  "dataVersion": "1.0"
}]
```

| Field | What it is | Example |
|-------|--------|---------|
| **id** | Unique identifier of the event | UUID |
| **eventType** | Event type — the most commonly used field for filtering | `BlobCreated`, `Contoso.OrderPlaced` |
| **subject** | Hierarchy of the affected resource or entity — allows prefix and suffix filters | `/blobServices/default/containers/images/blob.png` |
| **eventTime** | When the event occurred (ISO 8601) | `2026-07-28T10:00:00Z` |
| **data** | Payload with the event details — defined by the publisher | Free-form (JSON) |
| **dataVersion** | Version of the `data` schema | `"1.0"` |

> Maximum size per event: **1 MB**. Events are sent in arrays (batches), maximum **1 MB per batch**.

### Filtering in subscriptions

When you create a subscription you define which events you are interested in. There are three types of filter:

#### 1. Filter by `eventType`

The most basic. Only events of that exact type arrive.

```
Filter: eventType = "Microsoft.Storage.BlobCreated"
→ Pass:     BlobCreated
→ Don't pass: BlobDeleted, BlobRenamed
```

#### 2. Filter by `subject` (prefix and suffix)

The `subject` follows a path hierarchy. You can filter by how it starts or how it ends.

**Prefix (startsWith)** — filters by container:
```
subject startsWith: "/blobServices/default/containers/images"
→ Only blobs uploaded to the "images" container, ignoring the rest
```

**Suffix (endsWith)** — filters by file extension:
```
subject endsWith: ".png"
→ Only PNG files, regardless of the container
```

You can combine both:
```
subject startsWith: "/blobServices/default/containers/images"
subject endsWith: ".png"
→ Only PNG files inside the "images" container
```

#### 3. Advanced Filters

They allow filtering by any field of the `data` payload, with comparison operators.

| Operator | Example |
|----------|---------|
| `NumberGreaterThan` | `data.amount > 1000` → only orders over €1000 |
| `StringContains` | `data.region contains "EU"` |
| `StringBeginsWith` | `data.category beginsWith "premium"` |
| `BoolEquals` | `data.isUrgent = true` |

```
Practical example — subscription for large orders:
  eventType = "Contoso.OrderPlaced"  AND  data.amount > 500
→ Only OrderPlaced with an amount above €500
```

### Delivery model: how the event reaches the handler

#### Fan-out: each subscription receives its copy

The same event published to a topic reaches **all subscriptions** whose filters match it, independently. Subscriptions **do not compete with each other**.

```
Event published to Topic
        │
        ├── Subscription 1 (filter: BlobCreated) → push → Function A  ← receives its copy
        │
        ├── Subscription 2 (filter: BlobDeleted) → push → Logic App B  ← receives its copy
        │
        └── Subscription 3 (no filter)           → push → Webhook C   ← receives its copy
```

#### One subscription = a single handler endpoint

Each subscription points to a single destination. If you want three different handlers to react to the same event, you create three subscriptions.

The handler can scale internally (several instances of a Function), but that is transparent to Event Grid — it only knows the endpoint URL and makes the POST once.

#### Delivery guarantees

- **At-least-once**: an event may be delivered more than once — design your handler to be idempotent
- **Retries**: exponential backoff for **24 hours** if the handler fails
- **Dead Letter**: if all retries expire, the event goes to a Storage Account (requires explicit configuration)

### AZ-204 exam keys

- If you use Event Hub as the subscription destination instead of a direct webhook, you introduce latency — the consumer pulls, it does not receive a push. Wrong when the requirement says "immediately".
- Events are at-least-once → handlers must be idempotent.
- Dead lettering must be configured explicitly (Storage Account); retries last 24 hours with exponential backoff.

## Relations

- [[azure-event-hubs]] — high-throughput event ingestion where consumers pull; Event Grid instead pushes to handlers
- [[azure-functions]] — typical handler for Event Grid events
- [[azure-logic-apps]] — handler that orchestrates a flow in response to an event
- [[azure-service-bus]] — message queue for decoupling producer and consumer; Event Grid only notifies that something happened
- [[azure-blob-storage]] — common publisher of events (e.g. `BlobCreated`) and dead-letter destination

## Tags

#azure #az204
