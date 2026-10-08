---
domain: agentic-system-design
tags: [agentic-system-design, rag]
sources: [ai-agent-book/chapter3.md]
updated: 2026-09-21
---

# Contextual Retrieval

## What it is

A chunking technique developed by Anthropic that prepends an LLM-generated context prefix to each text chunk before indexing. The prefix anchors the chunk in its original document context — document title, section, entity, time period — resolving the ambiguity that standard chunking creates by severing surrounding context.

## What problem it solves

Standard chunking (fixed-size or recursive) severs closely related context. An isolated chunk like "The company's second-quarter revenue grew by 3%" is ambiguous at embedding time: which company? which year? which quarter? Without that context, the embedding misrepresents the chunk's true meaning, and retrieval accuracy drops — wrong company documents match, correct ones are missed.

---

## Notes

### How it works

During the **indexing phase** (not at query time):

1. For each text chunk, call an LLM with the full document as context
2. The LLM generates a short prefix summary: e.g., `[This text is from the 'Key Performance Indicators' section of ACME Corporation's 2025 Q2 Financial Report]`
3. Concatenate prefix + original chunk
4. Embed and index the combined text

The prefix is invisible to the user — it exists only to anchor the chunk's meaning in the vector space and the BM25 keyword index.

**Cost control**: use prompt caching (the full document as the cached prefix). Repeated chunk calls reuse the same document context at ~1/10 the token cost. Final cost: approximately **$1 per million document tokens**.

---

### Why it strengthens both retrieval modes simultaneously

| Retrieval mode | How the prefix helps |
|---|---|
| **Dense (vector embeddings)** | Prefix injects semantic background → embedding vector better represents chunk's true meaning → closer match to query intent |
| **Sparse (BM25 keyword)** | Prefix adds precisely matchable keywords ("ACME", "2025 Q2") → exact term overlap for specific queries |

This is one of the few techniques that improves both paths at once. Pair with hybrid retrieval (dense + BM25 + RRF) to compound the gains.

---

### Measured impact (Anthropic research)

| Combination | Retrieval failure rate reduction |
|---|---|
| Contextual Retrieval + BM25 | **–49%** |
| Contextual Retrieval + BM25 + reranker | **–67%** |

The reranker acts as a third-stage filter after fusion — it re-scores the fused candidates using cross-encoder reasoning, catching cases where the prefix + BM25 still surface near-misses.

---

### Critical distinction: not the same as Contextual Compression

Similar name, opposite operation, different phase:

| | Contextual Retrieval | Contextual Compression |
|---|---|---|
| **Phase** | Indexing (offline, pre-query) | Runtime (online, during a session) |
| **Target** | Text chunks in the knowledge base | Current session's conversation history |
| **Operation** | **Additive** — adds prefix context to chunks | **Subtractive** — trims irrelevant turns to save window space |
| **Goal** | Improve retrievability | Reduce context window consumption |

Confusing the two leads to applying the wrong technique at the wrong layer.

---

### Applied to user memory

Standard conversation-history chunking creates isolated chunks with no inter-chunk awareness. Applying contextual retrieval at indexing time generates prefixes like:

- `[Wife Patricia Thompson setting up initial wire transfer, January 2025]`
- `[Husband James Thompson modifying prior wire transfer, February 2025]`
- `[Wife modifying wire transfer again after husband's change, February 2025]`

Now the agent sees not just the content of each chunk, but **who said it, when, and in what intent context**. Factual contradictions across sessions become resolvable — the agent has the temporal and authorship metadata needed to determine which instruction is authoritative.

This is the fix for the multi-session contradiction problem in [[agentic-rag#Applying agentic RAG to user memory]].

---

### When to use

- Knowledge base has large documents (financial reports, legal filings, technical manuals) where chunks frequently lose their referential anchor
- Multi-hop queries where the right chunk must be contextually matched (not just keyword-matched) to a sub-question
- User memory systems where conversation chunks need temporal and entity context to resolve contradictions
- Anywhere hybrid retrieval is already in place and you want to maximize its effectiveness

Not worth the extra LLM indexing calls for:
- Small knowledge bases (< few hundred chunks) where full documents fit in context
- Queries that are already keyword-precise (no ambiguity about entity/time)

---

## Relations

- [[chapter3-agentic-rag]] — the retrieval orchestration layer that calls the knowledge base; contextual retrieval improves what gets retrieved per call
- [[chapter3-two-tier-memory-architecture]] — combines contextual retrieval (detail recall) with JSON Cards (overview in context) to reach L3 proactive service
- [[chapter3-user-memory-systems]] — the L2/L3 memory problems this technique directly addresses
- [[../rag/hybrid-search]] — dense + sparse + RRF fusion that contextual retrieval amplifies
- [[../rag/bm25]] — the sparse retrieval path that gets the biggest keyword improvement from context prefixes

## Tags

#agentic-system-design #rag
