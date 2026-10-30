---
domain: azure
tags: [azure, az204]
sources: []
updated: 2026-10-30
---

# Azure Cosmos DB

## What it is

Azure's globally distributed, schema-less database service. Data is organized in a four-level hierarchy (account → database → container → items), scaled through Request Units (RU/s) and distributed across physical partitions by a partition key. The API chosen at account creation (NoSQL, MongoDB, Cassandra, Gremlin, Table) decides how data is modeled and queried.

## What problem it solves

It gives applications a database with tunable consistency, global replication and throughput billed by operations rather than by CPU/memory. The API layer lets existing MongoDB, Cassandra or Table Storage applications migrate by changing only the connection string, while server-side features (change feed, stored procedures, triggers) cover reactive processing and ACID transactions inside a partition.

## Notes

### Structure in Azure

Cosmos DB has four hierarchical levels:

```
Account (az cosmosdb create)
└── Database
    └── Container  ←  equivalent to a table (no fixed schema)
        └── Items  ←  JSON documents
```

#### What is configured at each level

**Account** — infrastructure settings shared by everything below it:
- Region and global replication
- API (NoSQL, MongoDB, Cassandra, Gremlin, Table) — cannot be changed afterwards
- Default consistency level
- Network, firewall, backups

**Database** — logical grouping. Allows RU/s to be shared among its containers.

**Container** — unit of scaling. It has:
- Partition key — determines how data is distributed
- Indexing policy
- TTL (Time To Live)
- Dedicated RU/s (optional, overrides the RU/s shared at database level)

---

### Partition Key

Determines how Cosmos distributes data across physical partitions. A bad choice produces **hot partitions** (one partition overloaded).

**Rule:** many distinct values (high cardinality) + your frequent queries filter by it.

| Partition key | Problem |
|---|---|
| `status` (3 possible values) | Hot partition — 80% of writes go to "shipped" |
| `country` (10 countries, 90% Spain) | Uneven distribution |
| `customerId` (thousands of customers) | Even distribution ✓ |

---

### Throughput — RU/s

Cosmos charges for operations, not for CPU/memory. Unit: **Request Units (RU/s)**.

- 1 KB read ≈ 1 RU
- Writes cost more than reads

They are assigned at two levels (not at the account):
- **Database** → RU/s shared among all its containers
- **Container** → dedicated RU/s, guaranteed for that container

---

### Consistency levels

5 levels, from strongest to weakest. They are configured by default at the **account**. They can be lowered per individual request, never raised. **Default: Session.**

| Level | Ordering | Recency | Notes | Use case |
|---|---|---|---|---|
| **Strong** | ✓ | Always the latest | Waits for all replicas. High latency. Avoid in multi-region. | Critical data that demands full consistency |
| **Bounded Staleness** | ✓ | Within the bound (X versions or T seconds) | Bounded, configurable lag | Feeds with acceptable lag but in order |
| **Session** ← default | ✓ (your session) | Your own writes | The SDK stores a session token per write and sends it on every read. Other clients may see different versions. | Web apps where the user sees the result of their own actions |
| **Consistent Prefix** | ✓ | No | Replicas apply the log in strict order. You will never see B without having seen A. | Audit logs, histories |
| **Eventual** | ✗ | No | Minimum latency, maximum availability. Replicas converge over time. | Counters, metrics, likes |

---

### CLI commands

```bash
# Create account
az cosmosdb create --name my-account --resource-group my-rg --location eastus

# Create database
az cosmosdb sql database create --account-name my-account --name my-db --resource-group my-rg

# Create container
az cosmosdb sql container create \
  --account-name my-account \
  --database-name my-db \
  --name my-container \
  --partition-key-path /customerId \
  --resource-group my-rg
```

---

### Cosmos DB APIs

The API is chosen when the account is created and **cannot be changed afterwards** — it defines how data is indexed and structured internally.

Cosmos DB stores everything in its own internal format. The API is a compatibility layer that translates the external data model to the internal format and back. That is why each API keeps its own model from the developer's point of view.

| API | Data model | Query language | Use case |
|---|---|---|---|
| **NoSQL (Core)** | JSON documents | SQL-like | New projects on Azure. The native one, no penalty. |
| **MongoDB** | BSON documents | MQL (MongoDB Query Language) | Migrate an existing MongoDB app without rewriting queries |
| **Cassandra** | Rows with columns | CQL (Cassandra Query Language) | Migrate an app that uses Cassandra. Write-heavy, time series |
| **Gremlin** | Nodes and edges (graph) | Gremlin traversal | Complex relationships: social networks, recommendations, fraud |
| **Table** | Key-value entities | OData | Migrate an existing app that uses Azure Table Storage |

#### Practical rule for choosing an API

- You have **existing code** with MongoDB or Cassandra → use its API, you only change the connection string
- The problem has **complex relationships** between entities → Gremlin
- **New project on Azure** → always NoSQL

---

### Change Feed

Ordered log of all modifications (inserts and updates) of a container.

**By default it does not include deletes.** There is an alternative mode — *All versions and deletes* — that does include them (available since 2021).

#### Lease container

Read progress is persisted in a **separate lease container** (you create it). Inside there is one lease document *per physical partition* — it acts as a bookmark so no data is lost if the consumer fails.

#### Consumers and scaling

| Scenario | Behavior |
|---|---|
| Several instances of the **same** Function | They share the physical partitions (horizontal scaling) |
| **Different** Functions with different lease containers | Each one reads the full feed independently |

The second pattern is for fan-out: a purchase inserts an item → Function A sends an email → Function B updates inventory.

#### Change Feed Processor (SDK)

Alternative to the Azure Functions trigger for consuming the Change Feed directly from code. Where the Functions trigger abstracts everything away, the Processor gives you explicit control and is the usual model when the consumer is a long-running service (worker, microservice).

##### What problem it solves

The Change Feed is a log of changes per physical partition. If you have 10 partitions and 3 workers, someone has to decide who reads what, store each one's progress and redistribute if a worker goes down. The Change Feed Processor does all of that automatically via the **lease container**.

##### The 4 components in detail

| Component | Role |
| --- | --- |
| **Monitored container** | The container whose changes you listen to |
| **Lease container** | Persists progress per physical partition (bookmark) |
| **Host / Worker** | Your process (app instance) that consumes the feed |
| **Delegate** | The function with your business logic — receives the batch of changes |

**Monitored container** — the container you want to observe. Example: `orders`.

**Lease container** — a separate container that the Processor manages for you. Inside there is one lease document per physical partition with the position read so far:
```
leases:
  worker-A → read up to position 47 of partition 0
  worker-B → read up to position 31 of partition 1
  worker-C → read up to position 58 of partition 2
```
Without this bookmark, if the worker goes down and restarts, it would reprocess all changes from the beginning.

**Delegate** — your business function. It receives a batch of changes and does whatever you need:
```csharp
static async Task HandleChangesAsync(
    ChangeFeedProcessorContext context,
    IReadOnlyCollection<MyItem> changes,
    CancellationToken token)
{
    foreach (var item in changes)
    {
        // each item is a document that was inserted or updated
        Console.WriteLine($"Change detected: {item.Id}");
        await NotifyWarehouse(item);
    }
}
```

##### Scaling and failover example

You have an online store. Cosmos DB has 3 physical partitions in the `orders` container. Every time an order arrives, you want to notify the warehouse.

**Initial scenario — 1 worker:**
```
Partitions of "orders":
  P0: [order#1, order#4, order#7...]
  P1: [order#2, order#5, order#8...]
  P2: [order#3, order#6, order#9...]

Worker-A reads all 3 partitions → calls the delegate with each change
```

**Scaling — you add 2 more workers:**
```
The Processor redistributes automatically:
  Worker-A → P0
  Worker-B → P1
  Worker-C → P2
```
You configure nothing — the lease container coordinates who reads what.

**Failover — Worker-B goes down:**
```
The lease container detects that Worker-B stopped renewing its lease
  → Worker-A or Worker-C adopt P1
  → they continue from the position Worker-B had saved
  → no change is lost or processed twice*
```
*Processing is **at-least-once**: if the worker fails right before confirming the lease, the batch is reprocessed. The delegate must be idempotent if this is critical.

```csharp
var processor = monitoredContainer
    .GetChangeFeedProcessorBuilder<MyItem>(
        processorName: "my-processor",        // equivalent to the consumer group name
        onChangesDelegate: HandleChangesAsync)
    .WithInstanceName("instance-1")           // unique name of this worker
    .WithLeaseContainer(leaseContainer)
    .Build();

await processor.StartAsync();

// The delegate receives the batch when there are changes
static async Task HandleChangesAsync(
    ChangeFeedProcessorContext context,
    IReadOnlyCollection<MyItem> changes,
    CancellationToken token)
{
    foreach (var item in changes)
    {
        // process each change
    }
}
```

**`processorName` as a consumer group:**

| Scenario | Behavior |
|---|---|
| Several instances with the **same** `processorName` | They coordinate via the lease container and **share** the physical partitions → horizontal scaling |
| Instances with a **different** `processorName` | Each group reads the **full feed independently** → fan-out |

**Pull model** — manual alternative where you control when and from which point you read. Less common on the exam; just know that it exists.

#### Change Feed design patterns

Four official Microsoft patterns for using the Change Feed:

| Pattern | Key question | Example |
|---|---|---|
| **Real-time stream processing** | What is happening *right now* overall? | Online store: every time someone adds a product to the cart, a process reads the stream and updates a dashboard → "1,432 products have been added to carts today" |
| **Event sourcing** | How did I get to this state? What happened before? | Banking system: instead of storing only the final balance, every operation is stored as an event (deposit +€100, payment -€30...). The Change Feed lets you rebuild the balance at any point in the past |
| **Data movement** | How do I keep several systems in sync? | A document is created or updated in Cosmos DB → a function copies it automatically to Blob Storage → Power BI always has fresh data with no manual intervention |
| **Event computing and notifications** | What do I do *when* this specific event happens? | User signs up → document created in Cosmos DB → Change Feed detects it → Azure Function fires → verification email is sent |

---

### Stored Procedures

JavaScript code that runs **server-side** inside Cosmos DB.

#### Why they matter — ACID transactions

Stored procedures are the **only way to run ACID transactions** in Cosmos DB. Everything an SP executes is atomic: all or nothing.

#### Restrictions

- They only operate within **a single partition key** — they cannot cross partitions
- Language: JavaScript only
- They must be registered in the container before use

#### Invoking from the SDK

```python
# Python SDK
container.scripts.execute_stored_procedure(
    sproc="procedureName",
    params=[param1, param2],
    partition_key="partitionValue"
)
```

#### Triggers and UDFs also exist

| Type | When it runs | Typical use |
|---|---|---|
| Pre-trigger | Before saving | Enrich/validate the document |
| Post-trigger | After saving | Create derived documents (audit log) |
| UDF | Inside a SQL query | Custom logic in SELECT |

---

### Triggers

Pre and post triggers written in JavaScript that run server-side.

#### Critical rule

Triggers in Cosmos DB **do not fire automatically**. The client must invoke them explicitly on each SDK call. If you don't include them, they don't run.

```python
# You must specify them — otherwise they don't run
container.create_item(
    body=item_data,
    pre_trigger_include=["myPreTrigger"],
    post_trigger_include=["myPostTrigger"]
)
```

#### Pre-trigger

Runs before saving. It can modify the request body.

```javascript
function enrichOrder() {
    var context = getContext();
    var request = context.getRequest();
    var item = request.getBody();

    if (item.total > 1000) {
        item.category = "high_value";
    }
    item.createdAt = new Date().toISOString();
    request.setBody(item);  // replaces the body with the enriched one
}
```

#### Post-trigger

Runs after saving, inside the same transaction. It can read the saved document and create/modify other documents **in the same partition**.

```javascript
function createAuditRecord() {
    var context = getContext();
    var container = context.getCollection();
    var savedItem = context.getResponse().getBody();

    if (savedItem.category === "high_value") {
        var audit = {
            id: "audit_" + savedItem.id,
            type: "audit_log",
            userId: savedItem.userId  // same partition key, mandatory
        };
        container.createDocument(container.getSelfLink(), audit,
            function(err) { if (err) throw err; });
    }
}
```

#### Restrictions and transactional behavior

| Rule | Detail |
|---|---|
| Invocation | Explicit on each SDK call |
| Scope | Only the partition of the document in question |
| Language | JavaScript |
| Transaction | Pre + operation + post = all atomic |
| Failure | If the trigger throws an error, the whole operation is rolled back |

---

### AZ-204 exam keys

- **Partition key:** if a partition key is bad, the answer is almost always *low cardinality → hot partition*.
- **Change Feed trigger:** the Cosmos DB trigger in Azure Functions manages the lease container automatically (you configure the name).
- **At-least-once:** if the Function fails midway through a batch, the lease does not advance → the batch is reprocessed.
- The change feed can also be read with the SDK (Change Feed Processor or pull model).
- **Pattern trick:** if the scenario says *"notify"*, *"send"*, *"trigger when"* or *"react to"* → **Event computing and notifications** + **Cosmos DB triggered Function App**.
- **Triggers:** they never fire on their own; the client must list them on every SDK call (`pre_trigger_include` / `post_trigger_include`).
- **Transactions:** stored procedures are the only way to get ACID transactions, and only within a single partition key.

## Relations

- [[azure-functions]] — the Cosmos DB trigger consumes the Change Feed and manages its lease container
- [[azure-container-services]] — sibling Azure compute-side note for running workloads that talk to Cosmos DB
- [[azure-blob-storage]] — a typical data-movement target for the Change Feed

## Tags

#azure #az204
