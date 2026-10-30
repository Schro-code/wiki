---
domain: azure
tags: [azure, az204]
sources: []
updated: 2026-10-30
---

# Azure Service Bus

## What it is

Azure Service Bus is a **message broker** that decouples producers and consumers with guaranteed delivery, using queues (point-to-point) and topics with subscriptions (pub/sub).

## What problem it solves

Direct communication between services creates coupling: if the receiver goes down, the sender fails too. Service Bus decouples producers and consumers with guaranteed delivery.

```
Without Service Bus:
  API → Service B
        (if B goes down, the API fails and the message is lost)

With Service Bus:
  API → [Queue/Topic] → Service B
        (B can go down and recover without losing messages)
```

> **Reference use case (for the examples below):** online store. The customer places an order → the API sends it to Service Bus → several services process it independently.

## Notes

### Key concepts

#### Namespace

Root container. It is the URL of your instance: `mystore.servicebus.windows.net`. All resources (queues, topics) live inside a single namespace.

*Example: you create a namespace `mystore-bus` in Azure and define a queue for orders inside it.*

---

#### Queue — point-to-point

A message is consumed by **a single receiver**. It is equivalent to a Topic with a single Subscription shared by all consumers.

```
Queue:
  Producer → [Queue] → Consumer A  (only one receives each message)
                     → Consumer B
                     → Consumer C

Topic + 1 shared subscription:
  Producer → [Topic] → [Subscription] → Consumer A
                                      → Consumer B
                                      → Consumer C
```

*Example: when the customer buys, the API puts the order in the queue. The warehouse service reads it, reserves the stock and marks the message as completed. Nobody else sees it.*

---

#### Topic + Subscription — pub/sub

A message reaches **multiple subscribers simultaneously**.

```
Store (API) → [topic: order-created]
                  → Subscription "warehouse" → Warehouse Service (reserves stock)
                  → Subscription "email"     → Email Service     (sends confirmation)
                  → Subscription "invoice"   → Invoice Service   (generates PDF)
```

*Example: the same "order created" event triggers all three services at once, with none of them knowing about the others.*

**Key concepts of the Topic + Subscription model:**

- **Topics are independent of each other.** There is no connection between one topic and another — each one is an isolated channel with its own lifecycle.
- **Each topic can have multiple Subscriptions**, and each Subscription can have SQL or Correlation filters to receive only the messages it cares about.
- **Subscriptions do NOT compete with each other.** Each Subscription receives its own copy of the message. If a message arrives at the topic and you have no filters, all Subscriptions receive it → risk of duplicate processing if it is not planned for.
- **Inside a Subscription, consumers DO compete.** If several services read from the same Subscription, only one receives each message (competing consumers). This is what guarantees the order is processed exactly once.

```
Topic → Subscription A → Consumer 1  ← they compete: only one processes
                       → Consumer 2
                       → Consumer 3

        Subscription B → Consumer 4  ← receives its OWN copy (does not compete with A)
```

Each Subscription can have **filters** to receive only relevant messages:

```
Subscription "premium-orders"
  SQL filter: UserType = 'premium'   ← only receives orders from premium customers
```

---

#### Dead-letter Queue (DLQ)

Automatic subqueue where messages that fail or expire go. It lets you audit errors without losing messages.

*Example: the warehouse service fails 10 times trying to process an order (corrupt stock). Service Bus moves that message to the DLQ. The team can review it later without it being lost.*

A message goes to the DLQ when:
- It exceeds the maximum number of attempts (`MaxDeliveryCount`)
- Its `TimeToLive` expires without being processed
- The consumer explicitly calls `DeadLetterAsync()`

---

#### Session — guaranteed FIFO

Groups messages by `SessionId` and guarantees they are processed in **strict order** by a single consumer at a time.

*Example: an order has 3 steps (create, pay, ship). All three messages carry `SessionId="order-42"`. Service Bus guarantees they are processed in that order, never "ship" before "pay".*

```
Messages with SessionId="order-42":
  [create] → [pay] → [ship]   ← always in this order
```

---

### Filters on Subscriptions

Filters let a Subscription receive only the messages it cares about, not all those of the Topic.

```
Without filter:
  Topic "order-created" → Subscription "all"     ← receives everything

With filter:
  Topic "order-created" → Subscription "premium" ← only CustomerType = 'premium'
                        → Subscription "spain"   ← only Country = 'ES'
```

Filters are applied on the message's **ApplicationProperties**, not on the body:

```csharp
message.ApplicationProperties["CustomerType"] = "premium";
message.ApplicationProperties["Country"] = "ES";
message.ApplicationProperties["Amount"] = 150;
```

#### SQL Filter

The most flexible. SQL-like conditions over the message properties.

```sql
CustomerType = 'premium' AND Country = 'ES'
Amount > 100
```

#### Correlation Filter

More limited but faster (Azure optimizes it internally). It only compares exact equality on a field.

```
CorrelationId = "premium-order"
```

Use it when you only need exact equality. If you need `>`, `<`, `AND`, `OR` → use a SQL Filter.

#### True / False Filter

- **True Filter**: receives all messages (default behavior when creating a Subscription)
- **False Filter**: receives none — useful to temporarily disable a Subscription without deleting it

---

### Receive modes

| Mode | Behavior | When to use |
|------|----------|-------------|
| `ReceiveAndDelete` | Deletes the message when it is received | When occasional loss is acceptable |
| `PeekLock` | Locks the message; the consumer must **Complete** or **Abandon** | When delivery guarantee is critical |

*Example with PeekLock: the warehouse service receives the order → locks it → tries to reserve stock → if all goes well, it calls `Complete` (deleted). If it fails, it calls `Abandon` (returns to the queue to be retried).*

---

### Message states

A message goes through different states during its lifecycle:

```
[Sent] → Scheduled ──(time arrives)──→ Active → Completed
                                              → Dead-lettered
```

| State | What it means |
|-------|---------------|
| **Scheduled** | Stored but invisible to consumers — it is not time yet |
| **Active** | Available — consumers can receive it |
| **Deferred** | The consumer intentionally "parked" it for later |
| **Dead-lettered** | Failed too many times or expired — goes to the DLQ |

---

### Important message properties

| Property | What it is for |
|----------|----------------|
| `MessageId` | Deduplication — if the same order is sent twice, the duplicate is discarded |
| `CorrelationId` | Two different uses — see section below |
| `TimeToLive` | Expiry — if it expires without being processed, it goes to the DLQ |
| `ScheduledEnqueueTimeUtc` | Deferred send — the message stays in Scheduled state (invisible) until the given time, then becomes Active |
| `DeliveryCount` | Delivery attempts — if it exceeds the maximum, it goes to the DLQ |
| `SessionId` | Groups messages to guarantee strict FIFO |
| `SequenceNumber` | Unique number assigned by Service Bus — not set by the dev, used to identify messages |

```csharp
// Message that will not be visible for 2 hours
var message = new ServiceBusMessage("Order #42");
message.ScheduledEnqueueTime = DateTimeOffset.UtcNow.AddHours(2);
await sender.SendMessageAsync(message);
```

#### CorrelationId — two different uses

It is the same message property but with different intentions depending on the context:

**Use 1 — Traceability (request/reply)**

Links a reply to the original request. The responder puts in `CorrelationId` the `MessageId` of the message that originated the request.

```
Orders sends:
  MessageId    = "msg-001"
  Body         = "Charge €99"

Payments replies:
  CorrelationId = "msg-001"   ← "this is the reply to msg-001"
  Body          = "Charge OK"
```

Azure does nothing automatically — you are the one who assigns the value and reads it later to correlate.

**Use 2 — Filter on Subscriptions (CorrelationFilter)**

The producer puts a classification label in `CorrelationId` and the Subscription filters by exact equality on that value.

```
Producer publishes:
  CorrelationId = "High"      ← priority label
  Body          = "Order #42"

Subscription "HighPriorityOrders":
  CorrelationFilter: CorrelationId = "High"  ← only receives these
```

---

### Tiers

| | Standard | Premium |
|-|----------|---------|
| Topics | ✅ | ✅ |
| Maximum message size | 256 KB | **100 MB** |
| VNet / Private Endpoint | ❌ | ✅ |
| Price | Pay-per-use | Reserved capacity |

---

### Service Bus vs Event Grid vs Event Hubs

| | Service Bus | Event Grid | Event Hubs |
|-|-------------|-----------|-----------|
| **Model** | Messaging with delivery guarantee | Reactive events (push) | Stream ingestion (pull) |
| **Volume** | Thousands of msg/s | Thousands of events/s | Millions of events/s |
| **Retention** | Configurable (up to 14 days) | Does not retain | Days (configurable) |
| **Use case** | Workflows, orders, queued tasks | React to changes in Azure | IoT telemetry, logs |

> Quick rule: if you need **delivery guarantee and ordering** → Service Bus. If you need to **react to events** → Event Grid. If you need to **process a massive stream** → Event Hubs.

---

### Maximizing throughput: MessagingFactory and Batched Store Access

#### Context

Typical exam scenario: many senders (producers), few receivers (consumers). The goal is to maximize the number of messages processed per second.

#### MessagingFactory — one connection per process

To send messages, your code opens a **TCP connection** with the service. That connection has a cost: it is negotiated, authenticated, and consumes resources on both sides.

In the classic SDK, that channel is managed by a `MessagingFactory`. Each factory = one TCP connection. From it, you can create multiple senders that share the same channel.

```
❌ BAD: one factory per send (inside the same process)
Process → factory1 (TCP) → Service Bus
        → factory2 (TCP) → Service Bus   ← unnecessary overhead
        → factory3 (TCP) → Service Bus

✅ GOOD: one shared factory
Process → factory (TCP) → sender1 → Service Bus
                        → sender2 → Service Bus
                        → sender3 → Service Bus
```

**Rule:** a single factory per process. When you scale out (several instances), each instance is an independent process with its own factory.

```
[Instance 1]  →  1 factory → 1 TCP → Service Bus
[Instance 2]  →  1 factory → 1 TCP → Service Bus
[Instance 3]  →  1 factory → 1 TCP → Service Bus
```

In the modern SDK (`Azure.Messaging.ServiceBus`) the equivalent is `ServiceBusClient` — same idea, different name. The practice is the same: a single shared client per process.

```csharp
// ✅ GOOD: singleton client for the whole process
static ServiceBusClient _client = new ServiceBusClient(connectionString);

// Several senders share the same underlying connection
var sender1 = _client.CreateSender("orders-queue");
var sender2 = _client.CreateSender("payments-queue");
```

#### Batched Store Access

A broker feature (not a client one). When enabled, Service Bus groups multiple write operations to its internal storage instead of doing one write per message.

```
Without batching:
  Message 1 → [write] → disk
  Message 2 → [write] → disk
  Message 3 → [write] → disk

With batching:
  Message 1 ↘
  Message 2 → [write batch] → disk  ← a single I/O operation
  Message 3 ↗
```

This reduces write latency and significantly increases throughput in high-volume scenarios.

> **Note:** Batched Store Access introduces a minimal additional write latency (microseconds), but the throughput benefit far outweighs it when there are many senders.

---

### Connection methods

There are two ways to connect an application to Azure Service Bus:

| Method | Authentication | When to use |
|--------|----------------|-------------|
| **Connection String (SAS)** | Secret included in the string | Local development, apps without managed identity |
| **Managed Identity (Passwordless)** | Azure manages the identity | Production, apps running in Azure |

---

#### Connection String (SAS)

The string carries the password embedded. Just pass it to the client.

```csharp
var client = new ServiceBusClient(connectionString);
```

**General format:**

```
Endpoint=sb://<namespace>.servicebus.windows.net/;SharedAccessKeyName=<policy>;SharedAccessKey=<key>
```

##### Connection String components

| Parameter | Possible values | What it represents | Example |
|-----------|-----------------|--------------------|---------|
| `Endpoint` | Namespace URL | Address of the Service Bus. Always starts with `sb://` and ends in `.servicebus.windows.net/` | `sb://mystore.servicebus.windows.net/` |
| `SharedAccessKeyName` | Name of the SAS policy configured in Azure | Identifies which access policy is used and, therefore, what permissions the string has | `RootManageSharedAccessKey` · `SendPolicy` · `ListenPolicy` |
| `SharedAccessKey` | Key generated by Azure (Base64) | The actual password. Azure generates two per policy (primary/secondary) so you can rotate without downtime | `<root-key>` |
| `EntityPath` | Name of the queue or topic | **Optional.** Restricts the string to a specific entity. Without this field, the string applies to the whole namespace | `auction-status` · `orders` |
| `TransportType` | `Amqp` · `AmqpWebSockets` | Transport protocol. `Amqp` uses port 5671. `AmqpWebSockets` uses 443, useful if the firewall blocks 5671 | `AmqpWebSockets` |
| `Authentication` | `ManagedIdentity` | **Passwordless only.** Replaces `SharedAccessKeyName` and `SharedAccessKey`. There is no secret in the string | `ManagedIdentity` |

##### SAS policy permissions

| Permission | What it allows |
|------------|----------------|
| **Send** | Publish messages |
| **Listen** | Consume messages |
| **Manage** | Send + Listen + create/delete entities |

> `RootManageSharedAccessKey` has all three permissions. Use it only in development — in production, create policies with the minimum permission needed.

##### Complete examples

```
# Full access to the namespace (development only)
Endpoint=sb://mystore.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=<root-key>

# Send only to a specific topic (least privilege)
Endpoint=sb://mystore.servicebus.windows.net/;SharedAccessKeyName=SendPolicy;SharedAccessKey=<send-key>;EntityPath=orders

# Listen + WebSockets (firewall blocks port 5671)
Endpoint=sb://mystore.servicebus.windows.net/;SharedAccessKeyName=ListenPolicy;SharedAccessKey=<listen-key>;TransportType=AmqpWebSockets

# Passwordless — no secrets (production)
Endpoint=sb://mystore.servicebus.windows.net/;Authentication=ManagedIdentity;TransportType=Amqp
```

---

#### Managed Identity (Passwordless)

There are no secrets in the code. Only the namespace name and a credential are passed.

```csharp
var client = new ServiceBusClient(
    "mystore.servicebus.windows.net",
    new DefaultAzureCredential()
);
```

`DefaultAzureCredential` tries methods in order until it finds a valid one:

| Order | Method | When it applies |
|-------|--------|-----------------|
| 1 | Managed Identity | App running in Azure (App Service, Function, VM...) |
| 2 | Service Principal | Environment variables `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_CLIENT_SECRET` configured |
| 3 | Azure CLI | Local development — the dev is authenticated with `az login` |

---

### Code example (.NET SDK)

```csharp
// Send message
await using var client = new ServiceBusClient(connectionString);
ServiceBusSender sender = client.CreateSender("orders");

ServiceBusMessage message = new ServiceBusMessage("{ orderId: 42 }");
await sender.SendMessageAsync(message);

// Receive with PeekLock
ServiceBusReceiver receiver = client.CreateReceiver("orders");
ServiceBusReceivedMessage msg = await receiver.ReceiveMessageAsync();

Console.WriteLine(msg.Body.ToString());
await receiver.CompleteMessageAsync(msg); // processed OK → deleted
// await receiver.AbandonMessageAsync(msg); // fails → returns to the queue
```

---

### AZ-204 exam keys

**Which filter to use**

| You need... | Use |
|---|---|
| Exact equality on a field | Correlation Filter |
| Complex conditions (`>`, `AND`, `OR`) | SQL Filter |
| Receive everything | True Filter (default) |
| Disable without deleting | False Filter |

**Telling the two CorrelationId uses apart**

| | Traceability | Filter |
|---|---|---|
| **Who assigns the value** | The responder, with the ID of the original message | The producer, as a label for the message type |
| **What for** | Link a request with its reply | Route to the correct Subscription |
| **Who reads it** | The service that made the request | Service Bus (to decide which Subscription it goes to) |

If the question says *"CorrelationId is used as the priority value"* → they are using it as a classification label (filter), not for traceability.

**Choosing between the three services**

Quick rule: **delivery guarantee and ordering** → Service Bus; **react to events** → Event Grid; **process a massive stream** → Event Hubs.

**Throughput summary**

| Technique | Level | Effect |
|-----------|-------|--------|
| One factory/client per process | Client (SDK) | Avoids multiplying unnecessary TCP connections |
| Batched Store Access | Broker (Service Bus) | Groups writes to storage, increases throughput |

**Passwordless**

If the question says *"password less"* or *"no credentials in the code"*, the answer uses `Authentication=ManagedIdentity` in the string or `DefaultAzureCredential` in the SDK.

## Relations

- [[azure-event-grid]] — reactive event routing (push), contrasted with Service Bus's guaranteed-delivery messaging
- [[azure-event-hubs]] — massive stream ingestion, contrasted with Service Bus's broker model
- [[azure-logic-apps]] — a Service Bus message is a common trigger for a Logic App workflow
- [[azure-functions]] — Functions commonly consume Service Bus queues and topics
- [[microsoft-entra-id]] — Managed Identity and `DefaultAzureCredential` for passwordless connections
- [[azure-rbac]] — data-plane roles such as Azure Service Bus Data Receiver for access control

## Tags

#azure #az204
