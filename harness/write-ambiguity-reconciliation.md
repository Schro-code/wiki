---
domain: harness
tags: [harness]
sources: []
updated: 2026-08-26
---

# Write Ambiguity Reconciliation

## What it is

A pattern for handling the case where a write operation crosses a network boundary and the response is lost — leaving the outcome genuinely unknown before any observation.

## What problem it solves

When a write request is sent and the connection drops before the response arrives, three outcomes are possible:

1. The write **landed** — the remote system applied it
2. The write **did not land** — the remote system never received it
3. **Unknown** — the request was sent, but we cannot tell which of the above happened

Retrying the write is wrong if the write is not idempotent: a second request could create a duplicate, conflict, or corrupt state. Failing immediately loses real writes that did land.

The pattern reads back the resource after the failure to determine which outcome actually occurred.

## Notes

**Decision tree after an ambiguous network failure (timeout, transport error, invalid response):**

```
write failed with ambiguous error
        │
        ▼
   is there a stable ID to read back?
        │
   no ──┼── yes
        │         │
        ▼         ▼
 INDETERMINATE  read current state
 (e.g. creates)       │
                       ▼
              version = expected AND
              content matches proposal?
                       │
              yes ─────┼───── no
                       │              │
                       ▼              ▼
                   APPLIED    version = original AND
                              content unchanged?
                                       │
                              yes ─────┼───── no
                                       │              │
                                       ▼              ▼
                                  NOT-APPLIED    INDETERMINATE
```

**For creates specifically:**

A page creation has no stable ID until it succeeds. If the write outcome is unknown, there is no URL to read back. The outcome is always indeterminate — a human must check the remote system manually.

**For updates:**

The page ID is known before the write (it was read during proposal preparation). After the failure:
- Read the page at that ID
- If version = proposal_version + 1 AND content matches what was proposed → write landed
- If version = original_version AND content unchanged → write did not land
- If neither → a concurrent edit happened or the read also failed → indeterminate

**Indeterminate state:**

When the outcome cannot be determined, the target must be blocked from new writes until a human resolves it. A human checks the remote system out-of-band and calls `resolveIndeterminate(outcome)` to unblock.

**Why this is not a retry:**

Retrying re-executes the write. This pattern re-reads to observe the outcome. The write is never repeated. The distinction matters because:
- Retrying a create could produce two pages
- Retrying an update could increment the version twice if the first write landed
- Reading is always safe — it has no side effects on the remote system

**Relation to distributed systems concepts:**

This is a form of **read-your-writes consistency check** applied to the failure path. It is also related to **at-least-once vs exactly-once delivery**: this pattern attempts to enforce exactly-once semantics at the application layer when the transport layer cannot guarantee it.

## Relations

- [[pi-coding-agent]] — the harness where this pattern was first implemented (approval gate)
- [[ai-agent-harness]] — broader context of agent infrastructure patterns

## Tags

#harness
