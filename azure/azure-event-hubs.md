---
domain: azure
tags: [azure, az204]
sources: []
updated: 2026-10-30
---

# Azure Event Hubs

## What it is

A large-scale event ingestion service: producers publish events to an Event Hub, which distributes them across ordered partitions that independent consumer groups read from.

## What problem it solves

You need to receive millions of events per second (logs, telemetry, IoT, clicks) without the destination collapsing.

```
Without Event Hubs:
  App → calls the DB / analytics service directly
        (traffic spike → the destination goes down)

With Event Hubs:
  App → Event Hubs → consumer A (real-time analytics)
                   → consumer B (save to DB)
                   → consumer C (alerts)
```

## Notes

### Key concepts

| Concept | What it is |
|---------|--------|
| **Namespace** | Container for Event Hubs (like a messaging server) |
| **Event Hub** | A specific channel inside the namespace |
| **Partition** | Internal division — each partition preserves event order |
| **Consumer Group** | Independent view of the Event Hub — each consuming app has its own |
| **Retention** | Days events are kept (1–90 days depending on tier) |

### Request flow

```
┌────────────────────────────────────────────────────┐
│  1. Producers publish events to the Event Hub.     │
│     (apps, IoT devices, services, logs)            │
└──────────────────────┬─────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────┐
│  2. Event Hubs distributes the events across       │
│     partitions. Each partition preserves order.    │
│                                                    │
│   Partition 0  │░░░░░░░░│                          │
│   Partition 1  │░░░░░░░░│                          │
│   Partition 2  │░░░░░░░░│                          │
└──────────────────────┬─────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────┐
│  3. Each Consumer Group reads independently.       │
│     Consumer Group A → Stream Analytics            │
│     Consumer Group B → your .NET app               │
└────────────────────────────────────────────────────┘
```

### How it really works: partitions, Consumer Groups and coordination

#### A Consumer Group is not a filter

A Consumer Group **sees all events from all partitions** — there is no filter by type, content or topic. Each Consumer Group is simply an independent reader of the same complete stream.

> Key difference from Service Bus: in Service Bus a Subscription can filter messages. In Event Hubs that concept does not exist — all Consumer Groups see everything.

#### Each Consumer Group is completely independent

Each Consumer Group keeps its own **offset** (read position) per partition, stored in **Azure Blob Storage**. If one is slow, fails or restarts, it does not affect the others.

```
                    ┌─────────────────────────────────┐
                    │          EVENT HUB               │
                    │                                  │
  Events ─────────► │  Partition 0: [e1, e2, e3, e4]  │
                    │  Partition 1: [e5, e6, e7, e8]  │
                    │  Partition 2: [e9, e10, e11]     │
                    └─────────────────────────────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              │                     │                     │
       CG "analytics"         CG "billing"          CG "audit"
       (stopped at e3)        (stopped at e1)       (up to date, e11)
              │                     │                     │
       Blob: own              Blob: own              Blob: own
       offsets                offsets                offsets
```

Each Consumer Group saves its checkpoints in its own Blob Storage container — they share nothing.

#### Partitions define the maximum parallelism within a Consumer Group

Within a Consumer Group, **each partition can only be read by one consumer at a time**. There can be no overlap.

```
Consumer Group "analytics"  →  3 partitions  →  maximum 3 parallel readers

  Partition 0 ──► Reader A  ✅
  Partition 1 ──► Reader B  ✅
  Partition 2 ──► Reader C  ✅

  If you add a Reader D → it has no partition to read, it waits until one is free
```

If you have more instances of your app than partitions, the extra instances sit idle. That is why **the number of partitions is the most important scaling decision** and it is immutable once the Event Hub is created.

#### EventProcessorClient coordinates all of this automatically

In production you do not manage this by hand. `EventProcessorClient` does three things:

1. **Distributes the partitions** among all active instances of the Consumer Group (via leases in Blob Storage)
2. **Saves checkpoints** automatically so events are not reprocessed if an instance goes down
3. **Rebalances** if an instance joins or leaves — reassigns its partitions to the others

```csharp
// EventProcessorClient: each instance registers and receives partitions automatically
var processor = new EventProcessorClient(
    checkpointStore,   // Blob Storage — leases and offsets go here
    consumerGroup,
    connectionString,
    eventHubName);

processor.ProcessEventAsync += HandleEvent;    // business logic
processor.ProcessErrorAsync += HandleError;

await processor.StartProcessingAsync();
// From here on, coordination is automatic
```

#### SDK — main clients

```csharp
// Send events
EventHubProducerClient producer = new(connectionString, eventHubName);
using EventDataBatch batch = await producer.CreateBatchAsync();
batch.TryAdd(new EventData("my event"));
await producer.SendAsync(batch);

// Receive (simple)
EventHubConsumerClient consumer = new(
    EventHubConsumerClient.DefaultConsumerGroupName,
    connectionString, eventHubName);

// Receive with checkpointing (production)
EventProcessorClient processor = new(checkpointStore, consumerGroup, ...);
processor.ProcessEventAsync += HandleEvent;
processor.ProcessErrorAsync += HandleError;
await processor.StartProcessingAsync();
```

#### Summary of relationships

| Concept | Relationship with the others |
|---------|----------------------|
| **Partition** | Unit of ordering and parallelism. Defines how many simultaneous readers a CG can have |
| **Consumer Group** | Independent view of the complete stream. Has its own offsets per partition |
| **Offset / Checkpoint** | Read position of a CG in a specific partition. Stored in Blob Storage |
| **EventProcessorClient** | Coordinates which instance reads each partition within a CG. Manages leases and checkpoints |
| **Blob Storage** | Where checkpoints are persisted — allows resuming after a crash without losing events |

### Event Hubs Capture

Built-in feature that **automatically saves events** to persistent storage, with no extra code.

#### The problem it solves

Event Hubs retains events only for the configured time. If you need to keep them for historical analysis or auditing, Capture does it automatically.

#### How it works

```
Producers → Event Hub → Capture dumps automatically
                               │
                               ▼
                     Azure Blob Storage
                     or Azure Data Lake
                     (Avro format)
```

Capture dumps when **any** of these conditions is met (whichever happens first):

| Parameter | Range |
|-----------|-------|
| **Time window** | 1–15 minutes |
| **Size window** | 10–500 MB |

#### Output format

Files are saved in **Avro format** (binary, compact). Compatible with Spark, Databricks, Azure Synapse.

### Summary: Event Hubs vs Capture

| | Event Hubs | Event Hubs Capture |
|-|-----------|-------------------|
| **Purpose** | Real-time ingestion and distribution | Automatic persistence for historical analysis |
| **Needs code** | Yes (producers and consumers) | No — enabled in the Event Hub configuration |
| **Destination** | Consumer Groups (apps) | Blob Storage or Data Lake |

### AZ-204 exam keys

- If the question mentions "multiple instances", "high availability" or "not losing events after a failure" → `EventProcessorClient`.
- A Consumer Group is not a filter (unlike a Service Bus Subscription): every Consumer Group sees all events from all partitions.
- The partition count is immutable after creation and caps parallel readers per Consumer Group.
- Capture dumps on whichever comes first: time window (1–15 min) or size window (10–500 MB); output is Avro.

## Relations

- [[azure-event-grid]] — event routing by subscription with push delivery; Event Hubs is pull-based ingestion at scale
- [[azure-service-bus]] — message broker whose Subscriptions can filter, unlike Event Hubs Consumer Groups
- [[azure-blob-storage]] — stores consumer checkpoints/leases and is a destination for Capture
- [[azure-functions]] — `eventHubTrigger` runs a function when an event arrives

## Tags

#azure #az204
