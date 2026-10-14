---
domain: system-design
tags: [system-design]
sources: []
updated: 2026-09-01
---

# CAP and PACELC

## What it is

Two theorems that describe the fundamental trade-offs in distributed systems. CAP defines what guarantees a distributed system can make during a network failure. PACELC extends CAP to cover normal operation — where latency, not failure, is the bottleneck.

## What problem it solves

When data lives on multiple machines, those machines can lose contact with each other (a partition). CAP and PACELC force you to be explicit about what your system does in that situation — and in normal operation too.

## Notes

### CAP Theorem

Eric Brewer's 2000 theorem: a distributed system can guarantee at most **2 of these 3 properties**:

| Property | Meaning |
|---|---|
| **Consistency (C)** | Every read gets the most recent write — or an error. No stale data. |
| **Availability (A)** | Every request gets a response. No errors, but data may be stale. |
| **Partition tolerance (P)** | The system keeps working when nodes can't communicate. |

**The real-world collapse:** network partitions always happen. A cable gets cut, a pod evicted, a datacenter goes dark. You cannot opt out of P in any real distributed system. The actual choice is:

> When a partition happens: **return an error** (choose C) or **return possibly stale data** (choose A)?

This yields two real positions:

| Position | On partition | Examples |
|---|---|---|
| **CP** | Returns an error rather than stale data | ZooKeeper, etcd, HBase, Redis (sentinel) |
| **AP** | Returns stale data rather than an error | Cassandra, DynamoDB, CouchDB |

Single-machine databases (PostgreSQL, MySQL) sidestep the problem entirely — no distribution, no partition risk. They're sometimes labeled "CA" but it's misleading; they just don't face the choice.

### PACELC Theorem

Daniel Abadi's 2012 extension. CAP only describes partition behavior — which is rare. PACELC adds the normal-operation trade-off:

> **If Partition** → choose **A**vailability or **C**onsistency  
> **Else** (normal operation) → choose **L**atency or **C**onsistency

Even without a partition, replicating a write across nodes takes time. You can wait for all nodes to confirm (consistent, higher latency) or respond immediately and replicate in the background (lower latency, temporarily inconsistent).

| Position | During partition | Normal operation | Examples |
|---|---|---|---|
| **PA/EL** | Available (may return stale) | Low latency (don't wait for replication) | Cassandra, DynamoDB |
| **PC/EC** | Consistent (may reject) | High consistency, higher latency | ZooKeeper, HBase, VoltDB |
| **PA/EC** | Available | Consistent (waits for replication) | MongoDB (default) |
| **PC/EL** | Consistent | Low latency | Rare — hard to achieve |

PACELC matters more day-to-day than CAP. Partitions happen once a month. The latency vs consistency trade-off happens on every single request.

### CAP "Consistency" vs ACID "Consistency"

Same word, completely different concepts.

| | ACID Consistency | CAP Consistency |
|---|---|---|
| **Problem** | Data integrity rules | Replication lag |
| **Question** | "Is the data valid?" | "Is the data current across all nodes?" |
| **Scope** | Single database | Multiple nodes |
| **Violated by** | A transaction breaking a constraint | A node returning stale data |
| **Example** | `balance >= 0` always holds | Read from replica B after writing to node A returns the write |

Mental model: **ACID consistency = correctness. CAP consistency = freshness.**

A PostgreSQL replica is ACID-consistent (data is always valid by its rules) but not CAP-consistent (it may lag a few milliseconds behind the primary).

### Developer implications

**Know what your database chose.**

Cassandra is PA/EL — it will always respond and it won't wait for replication. Assuming every read is the latest write will produce bugs. ZooKeeper is PC/EC — it may reject requests during a partition rather than return stale data.

**Match the trade-off to the use case.**

| Use case | What you need | Database choice |
|---|---|---|
| Bank balance, inventory, seat booking | Correct answer, never stale | CP / PC/EC |
| Like counts, view counts, metrics | Fast response, eventual accuracy | AP / PA/EL |
| User profile, product catalog | Balance — AP is usually fine | AP / PA/EL |
| Distributed config, leader election | Strong consistency required | CP / PC/EC |

**AP systems require deliberate design.**

Picking an AP system means your code must handle the possibility that two reads to two different replicas return different values for the same key. Common patterns:

- **Last-write-wins** — timestamp writes, keep the latest on conflict
- **Quorum reads** — require N replicas to agree before returning (raises consistency, costs latency)
- **Idempotent writes** — design operations safe to run twice
- **Read-repair** — detect divergence on read, fix silently in the background

**The practical rule:** start with PostgreSQL. It avoids the whole problem. Move to distributed when you have a proven, measured scaling need — not because distributed sounds interesting.

## Relations

- [[acid-and-base]] — ACID databases are effectively CP; BASE databases are AP. CAP/PACELC is the theoretical framework explaining why that trade-off exists.
- [[ip-addresses]] — networking fundamentals that underpin the partitions CAP and PACELC describe

## Tags

#system-design
