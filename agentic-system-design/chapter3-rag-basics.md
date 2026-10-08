---
domain: agentic-system-design
tags: [agentic-system-design, rag]
sources: [ai-agent-book/chapter3.md]
updated: 2026-09-17
---

# RAG Basics — Knowledge Acquisition Pipeline

## What it is

Retrieval-Augmented Generation (RAG) is the standard pattern for giving an agent access to a knowledge base that lives outside the LLM's weights. The agent retrieves relevant fragments at query time and injects them into the prompt before generating an answer.

## What problem it solves

LLM training data has a cutoff date and no awareness of company-specific content. RAG lets the agent answer questions about documents that were never in training — policy manuals, legal codes, product FAQs — and keeps the knowledge base updatable without retraining the model.

## Notes

### The core flow

```
User query
    ↓
Retriever — finds top-K relevant fragments from the knowledge base
    ↓
Augment — query + retrieved fragments → full prompt
    ↓
LLM — generates answer grounded in the retrieved context
```

Three discrete engineering problems stack on top of each other: how to cut documents into retrievable pieces, how to match a query to a chunk, and how to search a large vector store fast.

---

### Problem 1 — Document chunking

Before retrieval is possible, documents must be cut into chunks small enough for an embedding model to represent accurately and for the context window to absorb selectively.

**Two failure modes:**

```
Too small → chunk loses its own context
  "Revenue grew by 3%" — which company? which quarter?
  The answer is not inside the chunk.

Too large → embedding vector gets diluted
  One chunk covers three topics. The embedding averages them.
  Retrieval finds it for none of them specifically.
```

**Default starting point**: 256–1024 tokens per chunk, 10–20% overlap between adjacent chunks. Tune from there against measured retrieval quality.

**Strategy choice by document type:**

| Your documents | Strategy |
|---|---|
| Markdown, HTML, structured docs | Recursive/structure-aware — production default |
| Unstructured text, speed matters | Fixed-size — simple and predictable |
| High-accuracy requirements, compute budget available | Semantic — split where embedding similarity drops |

- **Fixed-size**: cut every N tokens, with overlap. Ignores document structure — a table or code block can be split mid-way.
- **Recursive/structure-aware**: splits along natural boundaries (headings → paragraphs → sentences), falling back to smaller units if a chunk is still too long. Best fit for structured source material.
- **Semantic**: computes embedding similarity between adjacent sentences; splits where similarity drops sharply. Each chunk has a single primary theme. Higher quality at extra embedding compute cost.

**The flaw every strategy shares**: chunking severs a fragment from its surrounding context. A chunk extracted from page 40 of a report no longer knows which report it came from or what the preceding section established. This is chunking's inherent blind spot — addressed by Contextual Retrieval (see [[chapter3-contextual-retrieval]] when that page exists).

---

### Problem 2 — Matching a query to a chunk: dense embeddings

Dense embeddings convert text into a high-dimensional vector. Semantically similar texts map to nearby vectors. Similarity is measured by cosine similarity — the cosine of the angle between two vectors, ranging from 0 (unrelated) to 1 (identical meaning).

**Why this matters for agents:**

```
Query: "I want my money back"
Chunk: "Refund policy: full refunds within 7 days..."

→ No shared keywords. Cosine similarity is high.
  Dense retrieval finds this match. Keyword search misses it.
```

Dense retrieval bridges informal user language and formal document language. This is the same reason hybrid retrieval (dense + BM25) was needed for product knowledge in user memory systems — see [[chapter3-user-memory-systems]].

**Model generation matters:**

| Generation | Key limitation |
|---|---|
| Word2Vec / GloVe | One fixed vector per word — "bank" is the same in "river bank" and "investment bank" |
| BERT | Context-aware, but 512 token input limit — cannot embed long documents |
| BGE-M3 (2024) | Context-aware, multilingual, long text, outputs dense + sparse + multi-vector simultaneously |

For production today: **BGE-M3 or equivalent** is the default embedding model choice.

**Cosine similarity intuition** (optional):

```
"How to raise a cat"     → A = (0.9, 0.5, 0.1)
"Cat care guide"         → B = (0.8, 0.6, 0.1)
"Stock investment"       → C = (0.1, 0.1, 0.9)

cos(A, B) ≈ 0.99  → very similar
cos(A, C) ≈ 0.25  → very different
```

---

### Problem 3 — Searching a large vector store fast: ANN indexes

Brute-force similarity search (compare query against every stored vector) becomes too slow once the knowledge base reaches millions of chunks. ANN (Approximate Nearest Neighbor) indexes find close vectors without checking all of them.

Two mainstream algorithms:

| Feature | ANNOY (tree-based) | HNSW (graph-based) |
|---|---|---|
| Build speed | Fast | Slower |
| Memory usage | Low | Higher |
| Incremental updates | Not supported — requires full rebuild | Supported (periodic rebuilds recommended after many inserts) |
| Query accuracy | High | Extremely high |
| Best for | Static datasets — rarely updated | Live systems — new documents added continuously |

**Default for agentic systems**: **HNSW**. Agents that ingest new documents continuously (indexing user uploads, crawling new policies) need incremental inserts without full rebuilds. ANNOY is a better fit when the knowledge base is set once and rarely changed.

---

### Problem 4 — Exact keyword matching: sparse embeddings (BM25)

Dense embeddings understand semantics but fail on exact terms. A query for "HTTP-403" or "clause 232" may return general discussions about "server errors" or "criminal law" — close in meaning, wrong in precision. Sparse retrieval solves this with exact keyword matching.

A sparse embedding represents a document as a high-dimensional vector where most values are zero — only dimensions corresponding to words that actually appear in the document are non-zero. Retrieval = finding documents that share keywords with the query.

**BM25 — the production standard for sparse retrieval**

BM25 scores how well a document's keywords match the query, with two corrections over naive word counting:

1. **TF saturation** — the 10th occurrence of a word adds far less than the 1st. A document repeating "refund" 50 times isn't 50× more relevant.
2. **Length normalization** — a long document shouldn't rank higher just because it contains more words.

**Practical consequence**: BM25 excels at technical identifiers, proper names, article numbers, error codes — exactly the cases where users type a precise term and expect an exact match.

**SPLADE — the modern upgrade to BM25**

BM25 fails when query and document use different words for the same concept ("roban el coche" vs "sustracción del vehículo"). SPLADE solves this with learned query expansion: it keeps the sparse format but adds semantically related terms during indexing and retrieval.

```
BM25 sparse vector:   [roban=1.2, coche=0.8, 0, 0, 0, ...]
SPLADE sparse vector: [roban=1.2, coche=0.8, sustracción=0.6, robo=0.9, vehículo=0.5, ...]
```

SPLADE output is still a sparse vector — compatible with standard inverted indexes, fast at query time — but the weights are learned from (query, relevant document) training pairs, not computed from frequency statistics.

| | BM25 | SPLADE |
|---|---|---|
| Term weights | Frequency-based | Learned from training data |
| Query expansion | No | Yes — learned |
| Dynamic corpus | Recompute IDF on every insert | One forward pass per new document |
| Multilingual | Weak (English-tuned) | Yes (if trained on multilingual data) |
| Training required | No | Yes — needs labeled (query, doc) pairs |
| Speed | Fast | Fast (inverted index) |

**When to choose SPLADE over BM25**: multilingual corpus, domain-specific terminology (insurance, legal, medical), or corpus that grows continuously without IDF recomputation budget. Default to BM25 when no labeled training pairs exist.

**The structural blind spots:**

```
Dense (semantic)  → understands "kitty ≈ cat", "refund ≈ money back"
                    fails: "HTTP-403", "clause 232", "BGE-M3" (exact terms)

Sparse (BM25)     → finds exact keywords reliably
                    fails: query "kitty" never finds a document that only says "cat"
```

Neither engine is reliable alone. Each covers the other's blind spot — which is why you always combine them.

---

### Problem 5 — Combining both: hybrid retrieval

Run both engines, merge the results. The difficulty is that cosine similarity scores (0–1) and BM25 scores (0 to tens) are incomparable distributions. You cannot average them.

**State of the art — the full hybrid landscape (2024)**

```
RETRIEVAL METHODS (run in parallel, each produces a ranked list)
│
├── Dense — semantic meaning
│   ├── Single-vector: BGE-M3, E5-large, sentence-transformers  (standard)
│   └── Multi-vector:  ColBERT (MaxSim)                         (advanced — see below)
│
└── Sparse — keyword / exact match
    ├── BM25      standard baseline, English-tuned
    └── SPLADE    learned sparse, state of the art for multilingual/domain text

FUSION METHODS (merge ranked lists into one candidate pool)
├── RRF     rank-based, no score normalization needed  (default)
├── Linear  score-based, needs normalization and α tuning
└── Learned train a model to combine scores (overkill for most agents)

RERANKING (refine the top-N candidates from the fused pool)
├── Cross-encoder   e.g. BAAI/bge-reranker-v2-m3   standard
└── ColBERT MaxSim  token-level matching             best for negation / comparison queries
```

**Three-stage hybrid pipeline:**

```
Stage 1 — Parallel retrieval
  Query → Dense engine (top-K)    ← semantic understanding
  Query → Sparse engine (top-K)   ← exact keyword matching (BM25 or SPLADE)
  Both run simultaneously.

Stage 2 — RRF fusion (Reciprocal Rank Fusion)
  Problem: score scales are incomparable.
  Solution: discard scores entirely, use only ranks.
  score = Σ 1 / (60 + rank)   ← sum across both result sets
  Simple, robust, no calibration needed.
  Trade-off: discards the rich relevance signal in the original scores.

Stage 3 — Reranking
  Take top-N from the merged pool (~50 candidates).
  Reranker reads query + document together, word by word.
  Outputs a fine relevance score. Reorder and return final top-K.
```

**The bi-encoder vs cross-encoder distinction** — the most important concept in this section:

| | Bi-encoder | Cross-encoder |
|---|---|---|
| How it works | Encodes query and document separately, compares vectors | Concatenates query + document, reads them together |
| Speed | Fast — pre-compute document embeddings offline | Slow — must run per query-document pair at inference |
| Accuracy | Good for broad screening | Much better at fine relevance judgment |
| Used in | Retrieval stage (millions of docs) | Reranking stage (top ~50 candidates only) |

The analogy: bi-encoder = recruiter skimming resumes. Cross-encoder = interviewer in a 1:1 conversation. You cannot interview a million candidates, but you can interview the top 50 the recruiter shortlisted.

**Agent design consequence**: if the agent returns wrong answers on queries where the document exists in the knowledge base, the problem is usually ranking, not retrieval. Add a cross-encoder reranker before blaming the embedding model. Reranking does not replace fusion — fusion produces the unified candidate pool; reranking refines the ranking within it.

**ColBERT — when negation and comparison queries fail**

Standard dense embeddings collapse a document into one pooled vector. This averaging destroys token-level semantics — two sentences with opposite meanings can produce nearly identical vectors if they share the same words:

```
"The insurance does NOT cover windshield damage"
"The insurance DOES cover windshield damage"
```

Both contain the same dominant tokens. The `not` / `does` distinction gets diluted when pooled. A query about what is NOT covered retrieves both equally.

ColBERT keeps one vector per token instead of pooling. Similarity is computed via **MaxSim**: each query token finds its single best match in the document; the total score is the sum of those best matches. The negation token `not` stays as an independent vector — it cannot be averaged away.

| | Single-vector dense | ColBERT (multi-vector) |
|---|---|---|
| Representation | 1 vector per document | 1 vector per token |
| Negation handling | Poor — pooled away | Good — token preserved |
| Storage cost | Low | 758× more vectors per document |
| Latency | Fast | 2.6× slower (benchmark: 391ms → 1,031ms) |
| Best used as | Retriever over full corpus | Re-ranker on top-100 candidates |

**When to add ColBERT**: measure negation recall on real queries first. If it's a confirmed problem, add ColBERT as a re-ranker on the top-100 candidates from the hybrid retrieval stage (not as the primary retriever). Only Qdrant supports ColBERT natively — Azure AI Search, Vertex AI, and Pinecone require external infrastructure.

---

### Measuring retrieval quality

Tuning a multi-stage pipeline requires objective metrics. Measure these on a test set of queries with known correct answers:

| Metric | Question it answers | Priority for RAG |
|---|---|---|
| **recall@k** | Did the right document appear in the top-K results? | Highest — if it's not in the context, the LLM cannot use it |
| MRR | How high up was the first relevant hit? | Important when LLM attends mostly to the top of context |
| nDCG | Quality of the full ranked list? | Useful when injecting multiple chunks |

**Practical threshold**: measure recall@10 before shipping. Below ~80% means the retrieval pipeline is the bottleneck — no amount of prompt engineering will compensate for missing documents.

---

### Complete agent retrieval stack

```
User query
    ↓
Dense retrieval (BGE-M3 + HNSW)        ← semantic understanding
    +
BM25 sparse retrieval                   ← exact keyword matching
    ↓
RRF fusion                              ← merge without score calibration
    ↓
Cross-encoder reranking (top ~50)       ← fine relevance judgment
    ↓
Top-K chunks injected into context
    ↓
LLM generates answer
```

No stage is optional in production. Dense alone misses exact terms. Sparse alone misses synonyms. RRF alone discards relevance depth. Reranking alone has no candidates to work from.

---

### Decision guide

```
What are my source documents?
  Structured (Markdown, HTML) → recursive/structure-aware chunking
  Unstructured, speed priority → fixed-size chunking
  High accuracy, compute available → semantic chunking

What chunk size?
  Start: 256–1024 tokens, 10–20% overlap
  Tune: measure recall@10 on a test set, adjust

What embedding model?
  Default: BGE-M3 (multilingual, long text, dense + sparse outputs)
  Legacy: BERT if short text and monolingual

Does my knowledge base grow continuously?
  Yes → HNSW index (incremental inserts)
  No  → ANNOY (simpler, lower memory)

Do users query with informal language against formal documents?
  Always → run dense + BM25 in parallel, fuse with RRF

Which sparse method?
  English corpus, no training data → BM25
  Multilingual / domain-specific / growing corpus → SPLADE

Do I need high answer accuracy?
  Yes → add cross-encoder reranking on top-50 RRF candidates

Do queries include negation ("does NOT cover X") or multi-chunk comparison?
  Measured recall problem → add ColBERT MaxSim as re-ranker on top-100 candidates
  (requires Qdrant; otherwise use cross-encoder + external infrastructure)

Is my retrieval quality good enough?
  Measure recall@10 on a labeled test set
  Below ~80% → fix the retrieval pipeline before touching prompts
  For ranking quality: also measure MRR and nDCG
```

---

## Relations

- [[chapter3-user-memory-systems]] — hybrid retrieval (dense + BM25) was introduced there for product knowledge; this page explains the full pipeline in depth
- [[../rag/hybrid-search]] — landscape completo de hybrid search: todos los métodos de fusión, comparativa de sparse retrieval
- [[../rag/splade]] — SPLADE en profundidad: arquitectura, entrenamiento, ventajas sobre BM25 en español
- [[../rag/colbert]] — ColBERT en profundidad: MaxSim, benchmark de negación, two-stage approach
- [[../rag/bm25]] — BM25 en profundidad: la fórmula, saturación TF, normalización por longitud
- [[../rag/rag-fundamentals]] — pipeline RAG completo con comparativa de vector databases

## Tags

#agentic-system-design #rag
