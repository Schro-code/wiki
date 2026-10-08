---
domain: agentic-system-design
tags: [agentic-system-design]
sources: [ai-agent-book/chapter3.md]
updated: 2026-09-17
---

# User Memory Systems

## What it is

A persistent layer that lets an agent recall user-specific facts across sessions. Memory is not in-context learning — it is a dedicated post-session LLM call that extracts, filters, and structures facts into a separate store that survives context resets.

## What problem it solves

Without memory, every session starts blank. The agent cannot personalize, cannot avoid asking the same questions twice, and cannot anticipate needs based on past behavior. Memory is what separates a stateless chatbot from an assistant that knows you.

---

## Notes

### The two memory tiers

Every agent needs to distinguish two fundamentally different stores:

| Tier | Scope | Nature | When to use |
|---|---|---|---|
| **Trajectory** | Current session only | Append-only log: messages + tool calls + results, in order. Never rewritten. | Immediate context: "what did I just say," "what did the tool return" |
| **Long-Term Memory** | Cross-session, bound to user ID | Distilled facts. Repeatedly rewritten, merged, and pruned. Read/written via explicit tool calls. | Personalization across sessions |

The trajectory is a log. Long-term memory is an archive. They answer different questions.

---

### Evaluating memory: three levels

Before building, define what "good" means. These three levels give you a concrete target:

- **L1 — Basic Recall**: can the agent return a fact the user stated directly and unambiguously? ("My membership number is 12345.") This is table stakes.
- **L2 — Multi-Session Retrieval**: can the agent reason over facts from different sessions, entities, and time periods? When a user with two cars says "schedule maintenance for my car," the agent must find both and ask — not guess.
- **L3 — Proactive Service**: can the agent synthesize old memories to head off a problem the user didn't ask about? When a flight is booked, it surfaces the stored passport, notices it expires in 3 months, and warns without being asked.

Run your memory system against all three levels. Most production systems pass L1, struggle at L2, and rarely reach L3 without deliberate design.

---

### Four storage formats

Same fact, four representations. The format determines not just how you store — it determines what retrieval you need to build, how updates work, and which evaluation level you can realistically reach.

| Format | Structure | Retrieval needed | Update logic | Max eval level |
|---|---|---|---|---|
| **Simple Notes** | One atomic fact per entry | Pure vector search | Append new note; old ones become stale | L1 reliable, L2/L3 unreliable |
| **Enhanced Notes** | Full narrative paragraph per memory | Vector search on larger chunks | Rewrite the full paragraph on any change | L1 + some L2 |
| **JSON Cards** | Category → subcategory → key-value (`work.position.title`) | Vector search + deterministic key lookup | Update one key, no side effects | L1 + L2 reliably |
| **Advanced JSON Cards** | JSON Card + `backstory` + `person` + `relationship` | Vector search + key lookup + entity filter (`person = "father"`) | Same as JSON Cards, scoped to a person | L1 + L2 + L3 |

**Simple Notes** accumulate contradictions over time — "lives in Beijing" and later "lives in Shanghai" both sit in the store. Tier 1 compression is mandatory to prune them. **JSON Cards** avoid this: update one key, old value is gone. **Advanced JSON Cards** add the `person` and `relationship` fields, which is what lets the agent distinguish "my dentist" from "my father's cardiologist" and answer "book annual checkups for all family members."

**Practical split**: don't pick one format for everything. Run two paths in parallel:

```
High-value, low-volume, needs disambiguation → Advanced JSON Cards
  User preferences, key relationships, identity facts

High-volume, low-value, transient facts → Simple Notes
  Conversational details, recent activity
```

The format you choose also dictates how compression works — you cannot cluster Simple Notes the same way you cluster JSON Cards.

---

### What to store: cognitive types

Three types of content, each requiring a different retrieval mechanism and write strategy:

| Type | What it is | Agent example | Retrieval | Write strategy |
|---|---|---|---|---|
| **Semantic** | Stable facts abstracted from events | "User is vegetarian. Prefers window seats." | Vector similarity — standard RAG | Overwrite when the fact changes |
| **Episodic** | Specific events with context and time | "User booked ANA flight to Tokyo on 2025-01-15" | Temporal + similarity — needs a timestamp index | Append-only, never rewrite history |
| **Procedural** | Behavioral patterns and workflows | "Book flights: search direct → confirm seat → apply FF number → order meal" | Pattern trigger, not a query — injected into system prompt | Written rarely, only after observing the pattern many times |

Semantic and episodic live in the memory store. Procedural usually never gets there — it ends up hardcoded in the system prompt or as a few-shot example once you've observed the workflow enough times.

The cognitive type and the storage format are independent choices — you pick them on separate criteria. Format depends on engineering needs (cost, update frequency, disambiguation). Type depends on what kind of content it is (fact, event, workflow).

**When to add each type:**

```
Start with semantic memory only.

Do users ask time-anchored questions ("when did we last discuss X?")?
  Yes → add episodic memory + temporal retrieval index

Does the agent repeat the same multi-step workflow for this user?
  Yes → extract procedural, inject as system prompt guidance
```

---

### Critical distinction: user memory vs product knowledge

Not everything that looks like "procedural knowledge" belongs in user memory. The question is: **is this fact specific to this user, or is it the same for every user?**

| Information | Type | Where it belongs |
|---|---|---|
| "User prefers window seats" | User-specific semantic | User long-term memory |
| "User booked a flight on Jan 15" | User-specific episodic | User long-term memory |
| "User's usual booking flow: direct → seat → FF number" | User-specific procedural | System prompt injection |
| "To change your policy: go to My Area → Policies → Edit" | Product knowledge | RAG knowledge base — same for every user |
| "Your next payment date is visible at My Area → Billing" | Product knowledge | RAG knowledge base — same for every user |

"How do I change my policy?" is not something to store in user memory — the answer is identical for every customer. It belongs in a knowledge base retrieved with RAG. User memory only stores what is **unique to this individual**.

The retrieval challenge with product knowledge is that users ask informally ("where do I see my payment?") while documents are written formally ("payment schedule consultation procedure"). Semantic search alone fails here — the embeddings don't match. Use **hybrid retrieval: semantic + BM25 keyword search** to bridge the gap.

**Complete agent context at inference time:**

```
User question
     ↓
RAG over product knowledge base (hybrid: semantic + BM25)
     ↓ relevant procedures / FAQs
     +
User long-term memory retrieval (semantic / temporal)
     ↓ user-specific facts
     +
Trajectory (current session context)
     ↓
LLM answers with all three layers combined
```

If the product knowledge base is small (5–10 common procedures cover 80% of questions), skip RAG entirely and inject them directly in the system prompt.

---

### Production memory frameworks

**Mem0 v3 (2026)** — append-only writes, conflict-resolution at retrieval time:
- One LLM call extracts facts → ADD only. "Lives in Beijing" and later "moved to Shanghai" coexist as separate dated facts.
- At query time: fuses semantic similarity + BM25 + entity matching + temporal ranking.
- Why this matters: v2 used UPDATE/DELETE at write time — an incorrect update permanently discards history. v3 avoids that. Append-only is safer.
- Benchmark improvement: LoCoMo 71.4 → 92.5, LongMemEval 67.8 → 94.4.

**Memobase** — structured profile slots + event timeline:
- Developers define fixed slots (`basic_info.name`, `work.job_title`). Writing to a slot overwrites the old value — the structure itself is the compression.
- Event Memory handles time-anchored queries separately.
- Uses buffered batch processing: conversations accumulate until a threshold triggers one extraction pass. Low latency on reads because profiles are pre-organized.
- Trade-off: less flexible than Mem0 (you must define the schema upfront), but more predictable and cheaper to operate.

---

### Memory compression — three tiers

Memory grows unboundedly without pruning. A three-tier strategy:

1. **Tier 1 — Importance scoring**: score each memory and prune below a threshold. In practice, only two signals are worth the compute:
   - `access_frequency`: increment a counter on every read. Cheap and deterministic.
   - `recency`: `1 - (days_since_created / 90)`. Cheap and deterministic.
   - `emotional_intensity` and `uniqueness` require extra LLM calls or vector searches per entry — skip unless you have a compelling reason.

   The weights are not fixed. A medical agent weights uniqueness higher. A weather assistant weights recency higher. Tune against your L1–L3 evaluation set.

2. **Tier 2 — Clustering and summarization**: group similar memories, generate a representative summary per cluster ("User frequently asks about weather, especially rain"). Archive originals to secondary storage.

3. **Tier 3 — Abstraction**: convert repeated episodic memories into semantic or procedural rules ("Prefers cost-effective products and values user reviews"). This is the highest value step but requires the most LLM reasoning.

---

### Privacy: sanitize before the cloud

If logs contain PII (names, IDs, addresses, passwords), sending them to a cloud LLM for extraction exposes that data.

Pattern: run a local small model (e.g., Qwen3 0.6B via Ollama, CPU-runnable) as a sanitizer before any data leaves the perimeter. The local model detects and strips PII, returning a sanitized transcript. Only the clean text reaches the cloud LLM.

A hybrid approach scales better at high throughput: regex first for obvious patterns (card numbers, ID formats), local LLM only on what regex passes. LLM-based detection reaches >95% recall with fewer false positives than regex alone.

---

### Decision guide

```
Is this information specific to this user?
  No → it's product knowledge → RAG knowledge base (hybrid: semantic + BM25)
       Is the knowledge base small (< ~10 procedures)?
         Yes → inject directly in system prompt, no RAG needed
  Yes → it's user memory ↓

Does the agent need to remember it across sessions?
  No → trajectory only, no memory store needed.
  Yes ↓

Is the data volume low and does disambiguation matter?
  Yes → Advanced JSON Cards
  No ↓

Do you need partial updates without rewriting full paragraphs?
  Yes → JSON Cards
  No → Simple Notes

Do users ask time-anchored questions ("when did we last discuss X?")?
  → add Episodic Memory + temporal retrieval index

Does the agent repeat the same multi-step workflow for this user?
  → extract Procedural memory, inject as system prompt guidance

Does your domain have sensitive PII in conversation logs?
  → local sanitizer (Qwen3 0.6B via Ollama) before any cloud extraction call
```

---

## Relations

- [[overview]] — broader agentic system design patterns this fits into
- [[../rag/index]] — RAG is the retrieval mechanism once the memory store grows to thousands of entries
- [[../harness/index]] — the harness is where trajectory is maintained and memory tool calls are executed

## Tags

#agentic-system-design
