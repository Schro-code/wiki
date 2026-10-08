---
domain: agentic-system-design
tags: [agentic-system-design, rag]
sources: [ai-agent-book/chapter3.md]
updated: 2026-09-21
---

# Two-Tier Memory Architecture

## What it is

A memory design that combines **Advanced JSON Cards** (structured, always resident in context) with **Contextual RAG** (unstructured, retrieved on demand). Cards hold the global overview; RAG surfaces conversation details. Neither alone achieves L3 proactive service — only their combination does.

## What problem it solves

Two approaches individually fail at the highest capability level:

- **Resident context only (no retrieval)**: the context window caps out. As memory grows to thousands of entries, you cannot keep it all resident. Details fall off.
- **Retrieval only (no resident summary)**: the agent has access to details but no global overview. It cannot discover cross-session connections it doesn't know to search for. Proactive service requires knowing what might matter *before* a user asks.

L3 proactive service — surfacing a problem the user didn't know to ask about — needs both a global view and precise detail access simultaneously.

---

## Notes

### The two tiers

| Tier | Format | Location | Purpose | Capacity |
|---|---|---|---|---|
| **Overview** | Advanced JSON Cards | Always in context | Core facts, relationships, upcoming events — the "always-visible summary" | Small, curated. Grows slowly. Pruned regularly. |
| **Detail** | Raw conversations (contextual-retrieval indexed) | Knowledge base, retrieved on demand | Full conversation history, supporting evidence, exact original statements | Unbounded. Only relevant chunks retrieved per query. |

The overview tier answers "what do I know about this user at a glance?" The detail tier answers "give me the exact exchange where they said X."

---

### How it enables proactive service (L3)

Worked example — flight and passport scenario:

1. **Fact review (overview tier)**: Agent reads JSON Cards. Two relevant facts are resident: "Tokyo trip booked for January" and "passport expiry: February 18, 2025"
2. **Association reasoning**: The flight date is one month before passport expiry — a potential risk. This reasoning happens over the *overview*, without any retrieval.
3. **Detail verification (detail tier)**: Agent calls `search_user_memory("passport") + search_user_memory("Tokyo flight")` to confirm exact dates and context from original conversations.
4. **Proactive output**: "Your passport expires 3 weeks after your Tokyo trip. I strongly recommend expedited renewal."

Step 2 is the key: the agent discovers the connection *before* retrieving. Without the overview tier in context, the agent doesn't know to query for passport details when the user asks about their trip.

---

### Three-level evaluation map

| Level | What's needed | Architecture requirement |
|---|---|---|
| **L1 — Basic Recall** | Return a fact stated directly | Any storage + simple retrieval |
| **L2 — Multi-Session Retrieval** | Reason across sessions, entities, time | Contextual retrieval (resolves contradictions, adds temporal metadata) |
| **L3 — Proactive Service** | Surface a hidden cross-session risk unprompted | Overview tier in context (sees the connection) + detail tier on demand (confirms it) |

L2 and L3 each require a new architectural layer on top of the previous one. You cannot skip to L3 without building L2 first.

---

### Relationship to other memory concepts

The two-tier architecture is the convergence of two separate design tracks:

- **User Memory track**: Simple Notes → JSON Cards → Advanced JSON Cards. Each step improves structure, disambiguation, and update precision. [[chapter3-user-memory-systems]]
- **Knowledge Base RAG track**: basic chunking → hybrid retrieval → contextual retrieval. Each step improves what is retrieved per query. [[chapter3-contextual-retrieval]]

The two-tier architecture is where these tracks intersect. It's not a new technique — it's a specific combination of existing techniques that, together, achieve what neither achieves alone.

---

### Design principles

1. **Curate the overview tier aggressively** — not every fact belongs in JSON Cards. Only facts that the agent should notice *before a user triggers a query*. Clutter here dilutes the proactive signal.
2. **Never duplicate between tiers** — the detail tier holds the raw source; the overview tier holds the extracted key. Don't store full conversation chunks in JSON Cards.
3. **Cards update on events, RAG index updates on conversations** — they have different write cadences. Decoupling the write paths avoids consistency headaches.
4. **The detail tier covers contradictions** — [[chapter3-contextual-retrieval]] prefixes give each conversation chunk its temporal and authorship context, letting the agent determine which of conflicting instructions is authoritative.

---

## Relations

- [[chapter3-user-memory-systems]] — the L1/L2/L3 evaluation framework and JSON Card formats (overview tier)
- [[chapter3-contextual-retrieval]] — the indexing technique that makes the detail tier robust against ambiguity and contradiction
- [[chapter3-agentic-rag]] — the ReAct retrieval loop that orchestrates detail-tier lookups at query time
- [[chapter3-knowledge-organization]] — RAPTOR and GraphRAG as alternative backend options for the detail tier

## Tags

#agentic-system-design #rag
