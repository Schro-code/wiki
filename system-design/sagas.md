---
domain: system-design
tags: [system-design]
sources: []
updated: 2026-09-02
---

# Sagas

## What it is

A Saga is a pattern for managing distributed transactions across multiple independent services. Instead of one global atomic operation, it breaks the work into a **sequence of local transactions** — one per service — where each step commits immediately to its own database and has a corresponding **compensating action** that undoes it if a later step fails.

The name comes from a 1987 paper by Hector Garcia-Molina and Kenneth Salem, borrowing from Icelandic/Norse literary sagas — long epic stories told in self-contained chapters, each standing alone but contributing to the whole.

## What problem it solves

When multiple independent services must all succeed or all fail together, and you cannot use 2PC — because you don't control all nodes, they are third-party APIs, or locking across service boundaries is impractical — Sagas give you eventual consistency without global locking or blocking.

## Notes

### The design

Each step in a Saga pairs a forward action with a compensating action:

| Step | Forward action | Compensating action |
|---|---|---|
| 1 | Charge the customer's card | Refund the charge |
| 2 | Reserve the item in inventory | Release the reservation |
| 3 | Create the order record | Cancel the order |

**Happy path:** all steps succeed. The system moves forward, each step committing immediately to its own service.

**Failure path:** if step 3 fails, compensations run in reverse order — release inventory, then refund the charge. The system ends in a clean state as if the operation never happened.

The system is temporarily inconsistent between steps — a reader could observe the charge without the order existing yet. This is acceptable because the Saga always resolves to one of two final states: fully committed forward, or fully compensated backward.

### Two conditions that must hold

**1. Each step must be undoable.**
The compensating action must genuinely reverse the effect. Steps with irreversible side effects (sending an email, triggering a physical shipment) must be placed at the end — only execute them once earlier steps are confirmed.

**2. Compensating actions must eventually succeed.**
If a compensation fails (e.g. the refund API is down), keep retrying with exponential backoff — for minutes, hours, days — until it works. Never abandon a compensation. If it truly never succeeds, escalate to human intervention. The system must always converge to a consistent state.

### The coordinator problem

Something must track which steps have completed. The critical rule: **state is written to durable storage before every action**, not after. The log is the source of truth — the coordinator is just a process that reads it and executes the next step.

If the coordinator crashes, it restarts, reads the log, and continues from exactly where it left off. This is why a Saga coordinator crashing is not catastrophic — unlike 2PC, where a coordinator crash leaves nodes frozen indefinitely.

### Concrete implementation on Azure

**Azure Durable Functions** is the native Azure answer. It automatically persists the orchestration history to Azure Storage (Table + Blob) after every activity completes.

Structure:
- **Activity Functions** — each one does a single step (charge card, reserve stock, create order) or a single compensation (refund, release)
- **Orchestrator Function** — calls activities in sequence, handles failures, triggers compensations
- **Azure Storage** — full event log per saga instance, written automatically

**Replay-based execution:** when the orchestrator restarts after a crash, it re-runs the orchestrator code from the beginning. Activities that already completed return their stored result from the log — they do not execute again. The orchestrator fast-forwards to the current position and continues.

This means activity functions must be **idempotent** — calling them twice with the same inputs must produce the same result. If a crash happens during an activity before the result is logged, the orchestrator will call it again on restart.

**What the event log looks like:**

```
[saga-order-xyz]
  10:00:01  started
  10:00:02  charge_payment → SUCCESS  (payment_id: pay_abc)
  10:00:03  reserve_inventory → SUCCESS  (reservation_id: res_def)
  10:00:04  create_order → FAILED
  10:00:05  compensating: release_inventory → SUCCESS
  10:00:06  compensating: refund_payment → SUCCESS
  10:00:07  saga: ROLLED BACK
```

If the orchestrator crashes between lines 4 and 5, it restarts, replays the log to line 4, and starts compensation from there.

### Two coordination styles

**Orchestration** — a central orchestrator (a Durable Function, a state machine, a Temporal workflow) calls each service and handles the failure logic explicitly. Easier to trace and debug. Recommended default.

**Choreography** — no central coordinator. Each service listens for events and publishes the next one. The Saga state is implicit in the event chain. Scales better but harder to trace when something goes wrong.

### When to use Sagas vs alternatives

| Situation | Pattern |
|---|---|
| Microservices, third-party APIs, no shared lock | Saga |
| All steps are your own DB writes, incomplete records are harmless | Pending flag + batch cleanup |
| You control all nodes, need strict atomicity | 2PC |
| Strict atomicity, tolerate extra round-trip | 3PC |

## Relations

- [[distributed-transactions]] — 2PC and 3PC are the protocol-based alternatives. Sagas are the practical microservices alternative: no locking, eventual consistency.
- [[cap-and-pacelc]] — Sagas are AP by design: they return results without waiting for global consistency, accepting that the system may be temporarily inconsistent between steps.

## Tags

#system-design
