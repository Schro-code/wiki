---
domain: system-design
tags: [system-design]
sources: []
updated: 2026-09-01
---

# ACID and BASE

## What they are

Two consistency models for databases. ACID prioritizes correctness. BASE prioritizes availability and scale. They represent different answers to the same question: what guarantees does the database make when something goes wrong?

## What problem they solve

As systems grow — more data, more users, more machines — databases face a choice: stay perfectly consistent (slow, harder to scale) or stay available and accept temporary inconsistency (fast, scales globally). ACID and BASE are the two sides of that trade-off.

## Notes

### ACID

Designed for a single database acting as one source of truth. Used by relational databases (PostgreSQL, MySQL).

| Property | Meaning | What it protects against |
|---|---|---|
| **Atomicity** | All operations in a transaction succeed or all are rolled back | Partial writes |
| **Consistency** | Transaction brings DB from one valid state to another — constraints always enforced | Invalid data |
| **Isolation** | Concurrent transactions behave as if sequential — no dirty reads | Race conditions |
| **Durability** | Committed data survives crashes (write-ahead log) | Data loss |

**Bank transfer example:** debit A, credit B. If the server crashes between the two operations, atomicity rolls everything back. Neither account changes.

**Isolation levels** — isolation has four levels, each trading performance for more protection:

| Level | Protection | Default in PostgreSQL |
|---|---|---|
| `READ UNCOMMITTED` | Almost none | No |
| `READ COMMITTED` | Never reads uncommitted data | Yes |
| `REPEATABLE READ` | Same row read twice returns same value | No |
| `SERIALIZABLE` | Full sequential behaviour | No |

The default covers 90% of cases. You request stronger levels explicitly when you have specific race conditions.

### BASE

Appears when data is split across multiple machines (replicas, shards, nodes). Keeping copies in sync takes time — BASE accepts that reality.

| Property | Meaning |
|---|---|
| **Basic Availability** | The system appears to work most of the time |
| **Soft-state** | Replicas don't have to be mutually consistent at all times |
| **Eventual consistency** | Data will become consistent — just not immediately |

**Profile picture example:** user updates their picture. The write hits the Madrid replica instantly. London and New York replicas lag 50–120ms behind. A user in London sees the old picture briefly. Eventually all replicas sync. The system never stopped working.

ACID would lock all replicas until they agree — safe but slow. BASE keeps all replicas accepting reads and writes and lets them catch up asynchronously.

### ACID vs BASE — when to use each

| | ACID | BASE |
|---|---|---|
| **Data model** | Relational (SQL) | Distributed NoSQL |
| **Scale model** | One primary + read replicas | Multiple equal nodes |
| **Consistency** | Immediate, always | Eventual |
| **Availability** | May reject operations to stay consistent | Always available |
| **Use when** | Correctness is critical | Scale and uptime beat perfect consistency |
| **Examples** | Bank transfers, orders, payments | Like counts, timelines, product catalogs |

Real systems: PostgreSQL/MySQL (ACID), Cassandra/DynamoDB/Redis (BASE).

### How ACID databases scale

ACID doesn't mean stuck on one server. Common strategies:

**Vertical scaling** — bigger machine (more CPU, RAM, faster disk). Simple. Has a ceiling and gets expensive fast.

**Read replicas** — all writes go to one primary (ACID guaranteed), reads distributed across replicas. Replicas lag slightly but that's acceptable for most reads. This is the most common pattern in production.

```
                    ┌── Read replica (London)
Writer (Madrid) ────┼── Read replica (New York)
                    └── Read replica (Tokyo)
```

**Sharding** — split data across servers by a key (users A–M on server 1, N–Z on server 2). Each shard is fully ACID. Cross-shard transactions break ACID — so you design the data model to keep transactions within one shard.

**Connection pooling** — tools like PgBouncer multiplex thousands of app connections into a smaller pool of real DB connections. Removes a common bottleneck without touching the DB itself.

**The ceiling:** ACID systems hit a write ceiling (one primary can only write so fast) and struggle with cross-server transactions. BASE systems don't have this ceiling. Companies like Amazon and Google built BASE systems not because SQL is bad, but because at millions of writes per second globally, ACID guarantees became too expensive.

**The practical rule:** start with ACID. A well-tuned PostgreSQL with read replicas handles more traffic than most companies will ever need. Move to BASE only when you have a proven, measured scaling problem.

## Relations

- [[ip-addresses]] — networking fundamentals that underpin distributed systems where BASE is needed
- [[ipc-transports]] — how distributed database nodes communicate with each other

## Tags

#system-design
