---
domain: rag
tags: [rag]
sources: []
updated: 2026-09-07
---

# RAG — Retrieval-Augmented Generation

## What it is

A pattern that lets an LLM answer questions about private or domain-specific documents by retrieving relevant passages at query time and passing them as context — instead of baking knowledge into the model weights.

## What problem it solves

LLMs don't know your documents. You could pass all documents in every prompt, but token limits, cost, and latency make that impractical beyond a few pages. RAG solves this: find only the relevant fragments, send those.

**Concrete scenario used as reference throughout this page:** 11 Spanish insurance PDFs (car, home, motorbike). A customer asks *"¿Qué cristales cubre el seguro de rotura de lunas?"*. RAG finds 2-3 relevant chunks from 382 total and passes them to the LLM.

## Notes

### The pipeline

```
PDFs ──► chunk ──► embed ──► store in vector DB
                                      │
user query ──► embed ──► nearest-neighbor search ──► top-k chunks ──► LLM ──► answer
```

---

### 1. Chunking

You cannot embed an entire document as one vector — the embedding would average everything into a blob that represents nothing precisely. You need to split documents into smaller, indexable units.

**The core tension:** chunks too small lose context; chunks too large dilute the signal.

#### Strategies — from simple to complex

**Character / token count**
Split every N characters or tokens, no awareness of sentence boundaries. Fast, deterministic. Problem: cuts mid-sentence constantly, producing broken fragments.

**Sentence splitting**
Use a sentence boundary detector (spaCy, NLTK). Natural language units, clean boundaries. Problem: sentence length varies enormously — one sentence might be 10 words, the next 80. The resulting chunks are inconsistent in size and semantic density.

**Paragraph splitting**
Split on blank lines or `\n\n`. Works well for well-structured prose. Problem: legal and insurance PDFs don't have reliable paragraph breaks — a single article (coverage + exclusions + limits) can span several visual blocks with no clear separator. You can split coverage from its own exclusions.

**Recursive character splitting** *(popular in LangChain)*
Tries separators in priority order: paragraph → sentence → word → character. Falls back to the next level if chunks are still too large. Better than naive character splitting but doesn't solve the semantic coherence problem.

**Sliding window with overlap** ← *chosen for this project*
Fixed word count (e.g. 512 words) with an overlap between consecutive chunks (e.g. 64 words):

```
chunk 1: words   1 → 512
chunk 2: words 449 → 960    ← 64-word overlap with chunk 1
chunk 3: words 897 → 1408
```

The overlap means a sentence at a boundary appears in both adjacent chunks — retrieval has two chances to find it. Chunks are consistent in size, fast to compute, deterministic. Discard chunks below a minimum size (< 30 words) to eliminate headers, footers, and table labels that carry no semantic content.

**Semantic chunking**
Embed each sentence. When the cosine similarity between consecutive sentence embeddings drops sharply, that's a topic boundary — split there. Produces semantically coherent chunks regardless of word count. Problem: requires running the embedding model twice (once to find boundaries, once to index), is non-deterministic (embedding model updates change chunk boundaries), and is slow at scale.

**LLM-based chunking**
Give the document to an LLM and ask it to identify logical sections. Most intelligent option — understands article structure, coverage blocks, numbered lists. Problems: expensive (an LLM call per document), slow, non-deterministic, and must re-run when the document changes. Only viable for small, high-value document sets that change rarely.

#### How to choose

| Strategy | When it works | When it fails |
|---|---|---|
| Character/token | Homogeneous text, no structure | Structured docs, cuts sentences |
| Sentence | Clean prose, consistent sentence length | Legal text with very long sentences |
| Paragraph | Well-formatted docs | PDFs without paragraph structure |
| Sliding window | Most cases — good default | When document has very clear logical sections |
| Semantic | When topic coherence is critical | Large corpora, budget constraints |
| LLM | Small, high-value docs that rarely change | Scale, cost, non-determinism |

---

### 2. Metadata

After chunking, each chunk gets a metadata payload stored alongside its vector. Metadata enables **filtered search** — narrowing the search space before the vector comparison runs.

#### How to design quality metadata

**Start from user queries, not from what's easy to extract.**

Write 20 realistic queries your users would ask. Group them by the type of constraint they imply:

| User query pattern | Metadata field needed |
|---|---|
| "¿cuánto cubre el seguro de...?" | `monetary_values` |
| "¿qué dice el artículo 15?" | `article_references` |
| "¿qué coberturas tiene el hogar?" | `coverage_types` |
| "¿esto aplica a coche o moto?" | `category` |
| "¿es la versión actualizada?" | `version`, `document_date` |

If a field doesn't appear in any realistic query pattern, it probably doesn't belong in the metadata.

#### Field types

**Structural** — where the chunk comes from. Always include these.
- `document_id`, `category`, `document_type`, `version`, `document_date`
- Extraction: from filename patterns with regex

**Semantic** — what the chunk is about, extracted from content.
- `article_references` → regex: `Artículo\s+\d+`
- `coverage_types` → NLP or keyword list matching
- `monetary_values` → regex: `\d+[\.,]?\d*\s*€`

**Relational** — position within the document (useful for ranking or context expansion).
- `chunk_index`, `pages`

#### Quality test for metadata

Ask: *can I answer these queries using only metadata filters, without reading the chunk text?*

- "Give me all chunks from the car policy about monetary limits" → filter `category=automovil` + `monetary_values not empty` ✓
- "Find chunks that mention Article 15" → filter `article_references contains Artículo 15` ✓

If your metadata can't express these filters, your schema is incomplete.

#### Stats from the insurance corpus

382 chunks from 11 PDFs: 71 unique article references · 79 coverage types · 315 monetary values extracted.

---

### 3. Embeddings

An embedding is a vector of N numbers that encodes the semantic meaning of a text. Texts with similar meaning produce nearby vectors; unrelated texts produce distant vectors.

**Model choice matters.** For Spanish documents, you need a multilingual model — an English-only model produces poor vectors for Spanish text.

**`intfloat/multilingual-e5-large`** — 100+ languages, 1024 dimensions, trained on (query, passage) pairs.

#### The E5 prefix system

Dense embedding models are trained on paired examples. E5 specifically trains on pairs tagged as `query:` and `passage:`, so the model learns to map them to nearby regions of vector space even though they are structurally different (a question vs. a statement).

```python
# When indexing chunks
embed("passage: La cobertura de rotura de lunas incluye parabrisas, luneta trasera...")

# When embedding a user query
embed("query: ¿Qué cristales cubre el seguro de lunas?")
```

Without the prefix, the model still works but quality drops — you're sending text the model doesn't know how to classify into its learned representation space.

**Key finding about embeddings and databases:** retrieval quality is determined by the embedding model, not by the vector database. All 4 databases in the benchmark (Azure AI Search, Vertex AI, Pinecone, Qdrant) returned **identical Recall@10 = 0.817** on pure vector search with the same embeddings. Switching databases does not improve retrieval — switching models does.

---

### 4. Ground truth — how to measure retrieval quality

To measure whether your retrieval is good, you need to know the correct answer for each query. Labeling thousands of (query, relevant-chunk) pairs by hand is impractical.

**Solution: generate synthetic queries with an LLM, in reverse.**

```
Normal RAG:   chunk → retrieve → LLM answers query
Ground truth: chunk → LLM generates query → chunk_id is the correct answer
```

For each chunk, the LLM reads it and generates realistic questions a user might ask. The `chunk_id` is automatically the ground truth. 382 chunks × ~7 queries each = 2,690 queries, no human labeling.

**Five query types** to test different failure modes:

| Type | Example | Tests |
|---|---|---|
| `semantic` | "¿Qué cristales cubre el seguro de lunas?" | Pure vector search |
| `hybrid` | "cobertura Artículo 15 responsabilidad civil límite" | Keywords + semantic |
| `filtered` | "coberturas del seguro de hogar tipo garantías" | Metadata filters |
| `negation` | "¿Qué NO cubre el seguro de moto?" | Negation understanding |
| `comparison` | "¿Diferencia entre cobertura de coche y moto?" | Multiple relevant chunks |

**Why negation is hard for vector search:** embedding "el seguro cubre rotura de lunas" and "el seguro NO cubre rotura de lunas" produces nearly identical vectors — same words, same topic. Cosine similarity is high between them because the embedding encodes semantic topic, not logical operators.

---

### 5. Retrieval metrics

Given a query with a known set of relevant chunks, how good was retrieval?

All metrics are defined at a cutoff k — you only look at the top-k results returned.

---

#### Recall@k

*Of all relevant chunks that exist, how many did retrieval find in the top k?*

```
Recall@k = (relevant chunks in top-k) / (total relevant chunks)
```

**Example:** a query has 5 relevant chunks in the corpus. You retrieve top 10. Relevant chunks appear at positions 1, 3, 7, 10. One is not retrieved.

```
Recall@10 = 4 / 5 = 0.80
```

**For RAG, Recall is the priority.** Missing a relevant chunk means the LLM gives an incomplete answer. In insurance, missing a coverage clause can mislead a customer about their policy.

---

#### Precision@k

*Of the top-k results returned, how many are actually relevant?*

```
Precision@k = (relevant chunks in top-k) / k
```

**Same example** (4 relevant chunks found in positions 1, 3, 7, 10 out of 10 results):

```
Precision@10 = 4 / 10 = 0.40
```

Precision and Recall pull in opposite directions: returning more results increases Recall but decreases Precision (more noise). RAG tolerates lower Precision because the LLM can ignore irrelevant context — but too many irrelevant chunks dilute the answer and cost tokens.

---

#### MRR — Mean Reciprocal Rank

*How high up is the first correct result?*

For a single query:
```
Reciprocal Rank = 1 / (position of first relevant result)
```

MRR is the average reciprocal rank across all queries.

**Example:**
- Query A: first relevant result at position 1 → RR = 1/1 = 1.0
- Query B: first relevant result at position 3 → RR = 1/3 = 0.33
- Query C: first relevant result at position 10 → RR = 1/10 = 0.10

```
MRR = (1.0 + 0.33 + 0.10) / 3 = 0.48
```

MRR is low priority for RAG — if you pass top-10 to the LLM, it doesn't matter much whether the best chunk is at position 1 or 5. It matters more for search engines where users only read the first 1-2 results.

---

#### nDCG@k — Normalized Discounted Cumulative Gain

*How good is the full ranking, rewarding relevant results that appear higher?*

The "discounted" part means results at lower positions get less credit:

```
DCG@k = Σ relevance_i / log2(i + 1)   for i = 1 to k
```

Where `relevance_i` = 1 if the result at position i is relevant, 0 otherwise. `log2(i+1)` grows with position, so dividing by it penalizes relevant results found late.

**Example** (relevant at positions 1, 3, 7, 10):

```
DCG@10 = 1/log2(2) + 1/log2(4) + 1/log2(8) + 1/log2(11)
       = 1/1 + 1/2 + 1/3 + 1/3.46
       = 1.0 + 0.5 + 0.33 + 0.29 = 2.12
```

**IDCG@10** = the DCG you'd get if all 4 relevant results were at positions 1, 2, 3, 4 (ideal ranking):

```
IDCG@10 = 1/log2(2) + 1/log2(3) + 1/log2(4) + 1/log2(5)
        = 1.0 + 0.63 + 0.5 + 0.43 = 2.56
```

```
nDCG@10 = DCG / IDCG = 2.12 / 2.56 = 0.83
```

nDCG = 1.0 means perfect ranking. nDCG = 0 means no relevant result found.

**For RAG:** nDCG is less critical than Recall. Relevant chunks at position 8 vs position 2 cost a few extra tokens but both reach the LLM. nDCG matters more when you want to minimize the prompt size (e.g. only send top-3).

---

#### Summary table

| Metric | Question answered | RAG priority | Why |
|---|---|---|---|
| Recall@k | Did I find all relevant chunks? | **High** | Missing chunks = incomplete LLM answer |
| Precision@k | Are my results mostly relevant? | Medium | Noise dilutes the LLM context |
| MRR | Is the first correct result near the top? | Low | LLM reads all top-k anyway |
| nDCG@k | Is the full ranking ordered well? | Low | Position matters less when all top-k are passed |

---

### 6. Vector databases — where differences appear

All vector databases tie on pure vector search (same embeddings = same nearest neighbors). Differences appear in three scenarios:

#### Hybrid search

Combines sparse (keyword) and dense (vector) retrieval. Vector search captures semantic similarity but misses exact term matches. BM25/TF-IDF captures exact terms but misses paraphrases. Running both and merging results gets the best of each.

**Why the gain is especially large for insurance documents:** legal text uses controlled vocabulary — the same exact phrases appear verbatim across all documents (`responsabilidad civil`, `Artículo 15`, `600 €`). Writers use this deliberately for legal precision. When a user query contains one of these exact terms, keyword matching finds the right chunk directly, while vector search wastes positions on semantically similar but terminologically different chunks.

**How to merge two ranked lists: RRF (Reciprocal Rank Fusion)**

The problem: vector scores (cosine similarity, 0–1) and BM25 scores (term frequency statistics) are on incompatible scales. You can't add them directly.

RRF ignores scores entirely and only uses ranks:

```
RRF_score(chunk) = 1/(k + rank_vector) + 1/(k + rank_bm25)
```

Where `k = 60` is a constant that smooths the difference between rank 1 and rank 2.

Concrete example:

| Chunk | Vector rank | BM25 rank | RRF score |
|---|---|---|---|
| A | 1 | 8 | 1/61 + 1/68 = 0.0311 |
| B | 4 | 1 | 1/64 + 1/61 = 0.0321 |
| C | 2 | 2 | 1/62 + 1/62 = **0.0323** |

Final ranking: C → B → A. Chunk C wins because it ranked high in both systems — consistently good, not dominant in either. No normalization needed, no weights to tune.

**BM25 vs TF-IDF sparse vectors:** Azure uses BM25, a general-purpose algorithm designed and tuned on English corpora. It applies stemming and stop word removal — in English. Applied to Spanish, it doesn't stem Spanish words correctly: `cubre`, `cubierto`, `cobertura` are the same root but BM25 treats them as different terms. TF-IDF sparse vectors are trained on your specific corpus — they learn which terms actually matter in your documents, language-agnostically. For Spanish insurance text, the corpus-trained approach wins.

| Database | Hybrid support | Recall@10 hybrid |
|---|---|---|
| Azure AI Search | BM25 + RRF | 0.826 |
| Vertex AI | Sparse TF-IDF + dense + RRF | 0.945 |
| Pinecone Local | None (emulator limitation) | — |
| Qdrant | Sparse TF-IDF + dense + RRF | **0.950** |

Qdrant: pure vector 0.827 → hybrid 0.950. **+14.9% Recall@10** just from adding sparse matching.

#### Filtered search

Narrows the search space using metadata before the vector comparison runs. Instead of searching all 382 chunks, you constrain to e.g. `category = hogar` first, then vector-search only within those chunks. Critical for multi-product corpora — avoids semantically similar chunks from the wrong product surfacing as results.

**The danger of a filter system that is too strict:** if the filter rejects relevant chunks before vector search runs, those chunks are permanently lost. A strict filter can produce lower recall than no filter at all.

This is exactly what happened to Vertex AI. Its **token restricts** require exact string matching. The benchmark queries were generated by an LLM — which sometimes wrote `"Hogar"` instead of `"hogar"`, or `"vivienda"` instead of `"hogar"`, or `"seguro del hogar"`. Every mismatch silently discarded the entire chunk before vector search ran.

> **Vertex AI filtered Recall@10 = 0.131** — worse than pure vector search with no filter (0.817). The filter made retrieval 6x worse.

Rule: **a vector database with a bad filter system is worse than no filtering at all.** Test your filter system before relying on it.

| Database | Filter system | Recall@10 filtered |
|---|---|---|
| Azure AI Search | OData filters | **0.929** |
| Pinecone Local | $eq / $in operators | 0.896 |
| Qdrant | Payload filters (match, range) | 0.860 |
| Vertex AI | Token restricts (exact match) | 0.131 |

Azure wins because OData is expressive and forgiving — it supports `contains()`, `search.in()`, ranges, and case-insensitive matching. It was designed for structured querying by humans and LLMs alike.

#### Operations (CRUD + consistency)

Matters when documents are updated frequently.

CRUD latency p50: Qdrant **42ms** (WAL architecture) vs Vertex AI 315ms. Write-to-read consistency: Azure 1,692ms vs Qdrant 558ms.

---

### 7. ColBERT — late interaction embeddings

Standard dense embeddings collapse each chunk into one vector (pooling over all tokens). ColBERT produces one vector **per token** — ~758 vectors per chunk. At query time, each query token is compared against each document token and the maximum similarity scores are summed (MaxSim).

**Why it helps:** a single pooled vector averages everything, losing token-level semantics. Negation queries fail because "cubre" and "no cubre" produce similar pooled vectors. ColBERT preserves the "no" as its own token representation.

**E5 dense vs ColBERT (both in Qdrant):**

| Query type | E5 dense Recall@10 | ColBERT Recall@10 | Gain |
|---|---|---|---|
| Semantic | 0.885 | **0.953** | +7.7% |
| Negation | 0.710 | **0.921** | **+29.7%** |
| Comparison | 0.492 | **0.667** | +35.6% |
| Latency p50 | 391ms | 1,031ms | 2.6x slower |

Trade-off: ~2.6x more latency, ~758x more storage per document. Only Qdrant supports ColBERT natively (multivector collections with MaxSim comparator).

---

### When to use what

**The primary decision: do you need ColBERT?**

ColBERT costs 2.6x latency and ~758x storage. Only invest in it if negation queries ("¿qué NO cubre?") or cross-category comparison queries are a real, measured problem for your users. Don't add it speculatively.

- **Need ColBERT** → Qdrant (only database with native multivector + MaxSim support)
- **Don't need ColBERT** → Azure AI Search for most cases (cheapest, best filtered search, hybrid built-in)

| Use case | Choice | Why |
|---|---|---|
| Maximum quality, negation/comparison queries critical | Qdrant + ColBERT | Recall@10=0.953, +29.7% on negation |
| Good quality, lowest cost, already on Azure | Azure AI Search | $75/mo, OData filters (0.929), BM25 hybrid |
| Hybrid search, moderate budget, self-hosted | Qdrant + sparse+dense | Recall@10=0.950 hybrid, fastest CRUD |
| GCP ecosystem | Vertex AI + sparse+dense | Good hybrid (0.945) — avoid for filtered search |
| Rapid prototyping | Chroma or Qdrant local | Zero infrastructure, in-memory |

**Practical advice:** start with Azure or Qdrant dense+sparse. Measure negation recall on real user queries after launch. Only add ColBERT if you measure a real gap — not before.

## Relations

- [[lda-vectordb-benchmark]] — hands-on project that produced all benchmark data on this page
- [[graphrag/graph-construction]] — alternative to RAG: replaces vector search with a knowledge graph; better for global questions, worse for specific lookups

## Tags

#rag
