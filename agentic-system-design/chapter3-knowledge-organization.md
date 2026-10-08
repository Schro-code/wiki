---
domain: agentic-system-design
tags: [agentic-system-design, rag, graphrag]
sources: [ai-agent-book/chapter3.md]
updated: 2026-09-17
---

# Knowledge Organization — Beyond Flat RAG

## What it is

A set of indexing strategies that structure knowledge before retrieval — using tree hierarchies, entity-relationship graphs, or filesystem paradigms — so that retrieval can operate on organized, pre-abstracted material rather than raw document fragments.

## What problem it solves

Naive RAG drops raw documents into a vector store and retrieves by similarity. This is insufficient for two structural failure classes that no retrieval improvement can fix:

**Failure 1 — Aggregation blindness**: if 100 case documents are in the store and top-K retrieves 20, the model reasons from a biased sample. It cannot count or aggregate across the full corpus.

**Failure 2 — Missing boundary semantics**: individual case documents record outcomes ("Veteran John approved"), not rules ("only veterans and licensed professionals qualify"). The boundary statement doesn't exist in any single document — retrieval cannot surface what was never written.

The fix belongs at indexing time, not at retrieval time.

## Notes

### The core constraint

The model's attention mechanism is similarity-based soft retrieval. It is not a thinking engine that aggregates, generalizes, or derives boundary rules from individual cases. It can only pattern-match against what is in its context window.

**Design consequence**: compute must be invested at indexing time to extract, abstract, and structure raw knowledge. Pre-compute aggregates. Distill rules explicitly. Build hierarchy. The retrieval stage can only find what indexing prepared.

```
Naive RAG:    raw documents → vector store → retrieval → LLM
                                                ↑
                              model must reason from incomplete samples

Structured:   raw documents → LLM pre-processing → organized index → retrieval → LLM
                                    ↑
                     aggregate, distill rules, build hierarchy here
```

---

### Approach 1 — RAPTOR: tree hierarchy

RAPTOR (Recursive Abstractive Processing for Tree-Organized Retrieval) builds a knowledge tree bottom-up.

```
Root node           ← Global Summary (one node)
    ↑
Cluster summaries   ← LLM-written summary per cluster (middle layer)
    ↑
Text chunks         ← leaf nodes (original document fragments)
    ↑
Original documents
```

**How it works:**
1. Split documents into text chunks (leaf nodes)
2. Cluster semantically similar chunks — algorithm groups "SSE2 supports 128-bit integers" and "SSE4.1 adds string comparison" into the same topic cluster
3. LLM writes a higher-level summary for each cluster (parent node)
4. Recurse until a single root summary covers the entire knowledge base

**What retrieval gains**: any level of abstraction is now reachable. A macro-level query finds the answer in the root or cluster summary; a detail query drills down to the leaf chunk. No single flat retrieval needs to span both.

**Pattern fit**: queries that go broad-to-narrow — *"explain X"*, *"what is the architecture of Y"*, *"drill into Z from high level to detail"*.

---

### Approach 2 — GraphRAG: entity-relationship graph

GraphRAG models knowledge as a graph of entity-relationship-entity triples.

```
(Beijing, is capital of, China)
(Dr. Zhang-A, works at, Renai Stomatology Hospital)
(Dr. Zhang-A, department, Dentistry)
(Dr. Zhang-B, works at, Huashan Hospital)
(Dr. Zhang-B, department, Cardiology)
```

**LLM extracts** entities and relationships from text → community detection finds thematic clusters → the graph becomes traversable along edges.

**Two capabilities that no flat store can replicate:**

**Multi-hop reasoning** — "What is the address of my doctor's hospital?" resolves as:
```
User → Dr. Zhang-A → Renai Stomatology Hospital → Address
```
A flat store requires multiple independent retrievals plus LLM stitching. The graph traverses relationship edges directly.

**Entity disambiguation** — two "Dr. Zhang" are distinct graph nodes with different edges. The structure resolves the ambiguity without additional reasoning. This is different from word-sense disambiguation (where context-aware embeddings help); here the problem is distinguishing two real-world individuals with the same name.

**Known limitation**: triple extraction loses conditional logic and temporal dependencies.

```
"If it rains next week, I'll cancel my beach trip and go to the museum instead"

→ triples extracted:
   (user, plans, beach trip)
   (user, has backup plan, museum)

→ lost: the conditional, the temporal dependency, the causation
```

GraphRAG is reliable for factual, relational, entity-centric knowledge. It is not a substitute for natural language memory — use both.

**Pattern fit**: queries that are relational — *"how does A relate to B?"*, *"who is connected to X?"*, *"what affected Y?"*. Medical consultation, legal case analysis, contact/relationship management.

---

### RAPTOR vs GraphRAG — decision

| | RAPTOR | GraphRAG |
|---|---|---|
| Structure | Tree (hierarchy) | Graph (network) |
| Navigation | Top-down (concept → detail) | Lateral (entity → related entity) |
| Best for | "Explain X from high level" | "How does A relate to B?" |
| Retrieval path | Macro summary → drill down | Node → traverse edges → neighbors |
| Limitation | Poor at relational queries | Loses conditional / temporal logic in triples |

In production, combining both often outperforms choosing one — they solve different problems and their failure modes don't overlap.

**When to use structured indexing at all:**

```
Queries are "find the fragment with this info"?
  → Hybrid retrieval (dense + sparse + reranking) is sufficient. No structured index needed.

Queries require cross-document synthesis?
  → Add RAPTOR or GraphRAG.

Queries follow multi-hop chains or need entity disambiguation?
  → GraphRAG specifically.

Queries go from concept overview to specific details?
  → RAPTOR specifically.
```

Cost note: structured indexes require LLM calls at both build time and query time. Significantly higher cost and latency than hybrid retrieval. Invest only when queries actually need it.

---

### Approach 3 — Filesystem paradigm (OpenViking)

ByteDance's Volcano Engine proposes a third philosophy: map all knowledge into a virtual filesystem.

> **Local project**: `/Users/LDAZZH/Desktop/Proyectos/OpenViking/` — open-source implementation. See [[../labs/openviking|openviking lab]] for hands-on notes.

```
viking://
├── resources/        ← external docs, codebases, web pages
├── user/memories/    ← user preferences, habits
└── agent/
    ├── skills/
    └── memories/
```

`viking://` is a virtual URI — the framework decides whether to load from RAM, disk, or remote. The agent accesses knowledge through paths without knowing the physical location.

**Three abstraction layers per resource:**

| Layer | Size | Purpose |
|---|---|---|
| L0 (Summary) | ~100 tokens | One-sentence overview — judge directory relevance quickly |
| L1 (Overview) | ~2000 tokens | Core info + use cases — agent planning and decision-making |
| L2 (Full Text) | full | Complete original — loaded on demand only for deep analysis |

Each directory auto-generates `.abstract` (L0) and `.overview` (L1). If L0 signals irrelevance, L1 and L2 never load. Most queries resolve at L1. This is the same progressive-disclosure principle as Skills (Chapter 2): lightweight metadata first, full content only where needed.

**Why plain text + Git over a database:**
- Humans can read, edit, and correct the agent's knowledge directly
- Git provides version control and rollback
- Agent writes to a working branch; changes merge through a review workflow
- At session end, the system proposes writing updates to `user/memories/`

**The linking requirement — the easily missed detail:**

L0/L1/L2 handle vertical abstraction within a directory. Horizontal association between entries is a separate requirement that is easy to skip.

Without cross-references between files, the only navigation options are sequential scan or vector retrieval. No graph traversal, no concept-to-concept walking. The more knowledge accumulates, the harder the store becomes to navigate.

**The right model is Wikipedia**: every entry that mentions another concept links to it. Every directory has an index page. The agent can walk from one concept to its neighbors — file links providing some of the navigation power of GraphRAG without the triple extraction overhead.

**The practical catch**: this doesn't happen automatically. Weaker models just append new files without linking back to existing entries. The knowledge-writing prompt must explicitly require:
1. Before adding a new entry, retrieve related existing entries
2. Link to them from the new entry
3. Update the directory index page

This creates a bidirectionally reachable reference network. Skipping this step produces a disconnected pile of files that grows harder to retrieve from over time.

---

### How these approaches complement the retrieval pipeline

```
                     INDEXING TIME (offline)
                     ┌─────────────────────────────────┐
Raw documents   →    │ Pre-process: aggregate, distill, │  → Structured index
                     │ build tree / graph / filesystem   │     (RAPTOR / GraphRAG /
                     └─────────────────────────────────┘      filesystem with links)
                                                                        ↓
                     QUERY TIME (online)
                                                           Hybrid retrieval
                                                           (dense + sparse + RRF + reranking)
                                                                        ↓
                                                           Top-K chunks → LLM
```

Retrieval quality is bounded by indexing quality. If the structure is wrong upstream, no retrieval improvement downstream can compensate.

---

### Decision guide

```
Do my queries require aggregation or statistics over the full corpus?
  Yes → pre-compute and index the aggregate at build time (not retrievable otherwise)

Do my queries ask for boundary rules (who qualifies, what is excluded)?
  Yes → distill the rule explicitly at indexing time (individual cases don't contain it)

Do my queries go from high-level concept to specific detail?
  Yes → RAPTOR tree index

Do my queries follow multi-hop chains or need entity disambiguation?
  Yes → GraphRAG knowledge graph

Do my queries combine both patterns?
  → RAPTOR + GraphRAG together

Is my knowledge base navigable (can the agent walk from concept to concept)?
  No links or index pages → add cross-references, update index pages on every write
  Prompt must explicitly require linking — models don't do this spontaneously

Is structured indexing worth the cost?
  Benchmark: if hybrid retrieval (dense + sparse + reranking) recall is already good
  for your real queries, structured indexing adds cost without benefit.
  Invest only when queries demonstrably require cross-document synthesis.
```

---

## Relations

- [[chapter3-rag-basics]] — retrieval pipeline (dense + sparse + hybrid + reranking) that operates on the structured index built here
- [[chapter3-user-memory-systems]] — knowledge organization for user memory specifically; structured indexing applies to the product knowledge layer
- [[../rag/hybrid-search]] — the retrieval stage downstream of indexing
- [[../rag/graphrag]] — GraphRAG in depth (entity extraction, community detection, local vs global search)

## Tags

#agentic-system-design #rag #graphrag
