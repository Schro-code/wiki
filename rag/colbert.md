---
domain: rag
tags: [rag]
sources: []
updated: 2026-09-07
---

# ColBERT — Late Interaction Embeddings

## What it is

A retrieval model that keeps one vector per token instead of collapsing a document into a single pooled vector. Similarity between a query and a document is computed token-by-token using the MaxSim operator.

## What problem it solves

Standard dense embeddings pool all token representations into one vector. That averaging destroys token-level semantics — two sentences with opposite meanings can produce nearly identical vectors if they share the same words.

**The negation problem:**

> *"El seguro **no** cubre la rotura de lunas panorámicas"*
> *"El seguro **sí** cubre la rotura de lunas panorámicas"*

Both sentences contain the same dominant tokens (`seguro`, `cubre`, `lunas`). When pooled into one vector, the `no` / `sí` distinction gets diluted — the vectors end up very close in space. A query about what the policy does NOT cover retrieves both equally.

In the benchmark, this produced a **Recall@10 = 0.710** on negation queries with standard dense embeddings. ColBERT raised it to **0.921 (+29.7%)**.

## Notes

### How it works

**Standard dense (one vector per document):**
```
"El seguro no cubre lunas" → pool → [0.2, 0.8, 0.1, ...]   ← 1 vector, negation lost
```

**ColBERT (one vector per token):**
```
"El"       → [0.1, 0.3, ...]
"seguro"   → [0.4, 0.7, ...]
"no"       → [0.9, 0.1, ...]   ← preserved as independent vector
"cubre"    → [0.2, 0.8, ...]
"lunas"    → [0.5, 0.3, ...]
```

Each token keeps its own position in vector space. The `no` token is not averaged away.

### MaxSim — how similarity is computed

With N query tokens and M document tokens, computing similarity is no longer a single dot product. ColBERT uses **MaxSim**:

```
for each query token q:
    score(q) = max(cosine(q, d) for each document token d)

final_score = Σ score(q) for all query tokens
```

Each query token finds its single best match in the document. The total score is the sum of those best matches.

**Concrete example.** Query: *"no cubre lunas"* (3 tokens).

Document A: *"seguro no cubre lunas panorámicas"*
```
"no"     → best match: "no"     → 0.95
"cubre"  → best match: "cubre"  → 0.91
"lunas"  → best match: "lunas"  → 0.88
MaxSim = 0.95 + 0.91 + 0.88 = 2.74
```

Document B: *"seguro sí cubre lunas panorámicas"*
```
"no"     → best match: "sí"     → 0.21   ← low similarity, different meaning
"cubre"  → best match: "cubre"  → 0.91
"lunas"  → best match: "lunas"  → 0.88
MaxSim = 0.21 + 0.91 + 0.88 = 2.00
```

Document A scores 2.74 vs Document B's 2.00. The negation is now distinguishable.

### Computational cost

With 758 tokens per document (average in the insurance benchmark) and 8 query tokens:

```
Comparisons per document: 758 × 8 = 6,064
Comparisons across 382 documents: 6,064 × 382 = 2,316,448

vs standard dense: 1 × 382 = 382
```

This explains the benchmark results:
- Dense E5: **391ms p50** latency
- ColBERT: **1,031ms p50** latency (2.6x slower)
- Storage: **758x more** vectors per document

### Two-stage approach — retriever vs re-ranker

ColBERT was originally designed as a **re-ranker**, not a retriever:

1. **First pass (cheap):** standard dense retrieval → top-100 candidates
2. **Second pass (expensive):** ColBERT MaxSim over only those 100 candidates → final ranking

This gives ColBERT quality at a fraction of the cost — you run MaxSim on 100 documents, not thousands.

**The recall ceiling problem:** if a relevant chunk is not in the top-100 from the first pass, ColBERT never sees it. The first stage sets an upper bound on what re-ranking can achieve. You can reorder but not recover missed chunks.

| Approach | When to use |
|---|---|
| Single-pass ColBERT | Small corpus (< 10k chunks) — fast enough, no ceiling risk |
| Two-stage (dense → ColBERT re-rank) | Large corpus (> 100k chunks) — single-pass too slow, first-pass recall is high |

The insurance benchmark (382 chunks) used single-pass — latency was acceptable and there was no ceiling risk.

### Words vs tokens — context limit

ColBERT models have a maximum token context limit (e.g. `jinaai/jina-colbert-v2` supports 8,192 tokens). Chunks are sized in words, not tokens.

**Conversion:** 1 word ≈ 1.3 tokens on average. Spanish is morphologically rich (accents, conjugations) — tokenization tends to produce slightly more tokens per word than English.

```
512 words × 1.3 tokens/word ≈ 665 tokens
+ special tokens ([CLS], [SEP]) + Spanish overhead → ~758 tokens average
```

Tokens beyond the model's context limit are silently truncated — no vector is produced for them. Rule: keep chunk size in words × 1.5 comfortably below the model's context limit.

### Results in the benchmark

Model used: `jinaai/jina-colbert-v2` (1024 dims per token). Only Qdrant was tested — it is the only database in the benchmark with native ColBERT support (multivector collections + MaxSim comparator).

| Query type | E5 dense Recall@10 | ColBERT Recall@10 | Gain |
|---|---|---|---|
| Semantic | 0.885 | **0.953** | +7.7% |
| Negation | 0.710 | **0.921** | **+29.7%** |
| Hybrid queries | 0.827 | **0.955** | +15.5% |
| Comparison (multi-chunk) | 0.492 | **0.667** | +35.6% |
| Latency p50 | 391ms | 1,031ms | 2.6x slower |

Negation and comparison queries benefit most — exactly the cases where pooled embeddings lose token-level semantics.

### When to use ColBERT

Don't add it speculatively. Measure first.

1. Launch with standard dense or dense+sparse hybrid
2. Collect real user queries and measure negation recall
3. If negation/comparison recall is a measured problem → add ColBERT

ColBERT is only supported natively by Qdrant (multivector + MaxSim). Azure AI Search, Vertex AI, and Pinecone require one dense vector per document — ColBERT is not possible on those without external re-ranking infrastructure.

## Relations

- [[rag-fundamentals]] — full RAG pipeline; ColBERT fits as an alternative embedding strategy in the retrieval stage
- [[lda-vectordb-benchmark]] — the project that benchmarked ColBERT against E5 dense on insurance documents

## Tags

#rag
