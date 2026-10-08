---
domain: rag
tags: [rag]
sources: []
updated: 2026-09-07
---
****
# Hybrid Search

## What it is

A retrieval strategy that combines two complementary systems — dense (semantic) and sparse (keyword) — and merges their results with a fusion algorithm. Neither system alone is optimal; together they cover each other's blind spots.

## What problem it solves

**Dense retrieval alone** finds semantically similar chunks but misses exact term matches. A query containing `Artículo 15` or `600 €` may return chunks about liability in general instead of the specific article.

**Sparse retrieval alone** finds exact keyword matches but misses paraphrases and synonyms. A query asking *"¿qué cubre el seguro?"* may miss a chunk that uses `cobertura` instead of `cubre`.

Hybrid search captures both: +14.9% Recall@10 on the insurance benchmark (Qdrant: 0.827 dense → 0.950 hybrid).

## Notes

### The landscape

```
HYBRID SEARCH
│
├── RETRIEVAL METHODS (each produces an independent ranked list)
│   │
│   ├── Dense — semantic meaning
│   │   ├── Single-vector: E5, BGE, sentence-transformers  → [[rag-fundamentals]]
│   │   └── Multi-vector:  ColBERT (MaxSim)                → [[colbert]]
│   │
│   └── Sparse — keyword / exact match
│       ├── TF-IDF    classical, not learned               → [[tf-idf]]
│       ├── BM25      improved TF-IDF, standard baseline   → [[bm25]]
│       └── SPLADE    learned sparse, state of the art     → [[splade]]
│
└── FUSION METHODS (merge the ranked lists into one final ranking)
    ├── RRF     rank-based, no score normalization needed   → see below
    ├── Linear  score-based, needs normalization            → see below
    └── Learned train a model to combine scores            → see below
```

---

### Dense vs Sparse — the key difference

| | Dense vector | Sparse vector |
|---|---|---|
| Dimensions | ~1024 (all filled) | 30,000+ (mostly zeros) |
| What each dim represents | Abstract learned feature | Roughly a word in the vocabulary |
| Captures | Semantic meaning, synonyms, paraphrases | Exact word presence and frequency |
| Fails on | Exact technical terms, article numbers | Paraphrases, synonyms, different phrasing |
| Example models | E5-large, BGE, sentence-transformers | TF-IDF, BM25, SPLADE |

**Dense:** `[0.23, -0.87, 0.41, 0.12, ...]` — 1024 non-zero values, meaning is distributed.

**Sparse:** `[0, 0, 4.2, 0, 0, 1.7, 0, 3.1, 0, ...]` — 30k dimensions, ~20 non-zero (the words in the chunk).

ColBERT is still dense — each token vector is 1024 non-zero dimensions. Multi-vector ≠ sparse.

---

### Sparse retrieval methods

#### TF-IDF
→ [[tf-idf]]

Classical baseline. Score = how often a term appears in the document (TF) × how rare it is across the corpus (IDF). Rare terms that appear frequently in a document get high scores. No learning, no parameters.

#### BM25
→ [[bm25]]

The standard baseline for keyword search (used in Elasticsearch, Lucene, Azure AI Search). Improves on TF-IDF with two additions:
- **Term saturation:** the 10th occurrence of a word adds less than the 1st (diminishing returns)
- **Length normalization:** a term in a short document is more significant than the same term in a long document

Tuned on English corpora — performs worse on morphologically rich languages like Spanish.

#### SPLADE — state of the art
→ [[splade]]

**Sparse Lexical and Expansion model.** A learned sparse model that combines the interpretability of BM25 with the semantic understanding of dense models. Trained end-to-end on (query, relevant doc) pairs. Key advantages:
- Learns which terms are important for retrieval (not just frequency)
- Performs **query expansion** — adds related terms that don't appear in the original text
- Produces sparse vectors that work with inverted indexes (fast at scale)
- Language-agnostic when trained on multilingual data

---

### Fusion methods

#### RRF — Reciprocal Rank Fusion

Rank-based fusion. Ignores raw scores entirely — only uses the position (rank) of each result in each system's ranked list.

```
RRF_score(chunk) = 1/(k + rank_dense) + 1/(k + rank_sparse)
```

Where `k = 60` smooths the difference between adjacent ranks.

**Concrete example:**

| Chunk | Dense rank | Sparse rank | RRF score |
|---|---|---|---|
| A | 1 | 8 | 1/61 + 1/68 = 0.0311 |
| B | 4 | 1 | 1/64 + 1/61 = 0.0321 |
| C | 2 | 2 | 1/62 + 1/62 = **0.0323** |

Result: C → B → A. Chunk C wins — consistently good in both, not dominant in either.

**Why RRF over score-based fusion:**
- Dense scores (cosine similarity, 0–1) and sparse scores (BM25, unbounded) are on incompatible scales
- Normalizing them requires knowing the score distribution in advance
- RRF needs no normalization, no hyperparameter tuning, no per-query calibration
- Empirically competitive with learned fusion on most benchmarks

#### Linear weighted fusion

```
final_score = α × dense_score + (1 - α) × sparse_score
```

Requires normalizing both score distributions first. The weight `α` needs tuning per corpus and query type. More fragile than RRF but can outperform it when the optimal `α` is known.

#### Learned fusion

Train a small model (or use a cross-encoder) to predict the optimal combination given the query and the candidate scores. Best quality, highest complexity. Used in production at search engines — overkill for most RAG applications.

---

### Results in the benchmark

Same corpus (382 chunks), same embeddings (multilingual-e5-large), only the retrieval strategy changes:

| Database | Strategy | Recall@10 |
|---|---|---|
| All 4 | Pure dense | 0.817 |
| Azure AI Search | Dense + BM25 + RRF | 0.826 |
| Vertex AI | Dense + sparse TF-IDF + RRF | 0.945 |
| Qdrant | Dense + sparse TF-IDF + RRF | **0.950** |

Azure uses BM25 (English-tuned). Vertex AI and Qdrant use corpus-trained sparse TF-IDF vectors — language-agnostic, better for Spanish.

---

### Choosing a strategy

| Use case | Strategy |
|---|---|
| Simple corpus, English | Dense + BM25 + RRF |
| Multilingual or domain-specific text | Dense + SPLADE (or sparse TF-IDF) + RRF |
| Maximum quality, latency budget | Dense + SPLADE + RRF + ColBERT re-rank |
| Prototyping | Dense only — add sparse once you measure the gap |

**Rule:** always start with dense only. Measure Recall on real queries. Only add sparse when you can quantify the improvement — hybrid adds infrastructure complexity.

## Relations

- [[rag-fundamentals]] — the full RAG pipeline; hybrid search is one component of the retrieval stage
- [[colbert]] — multi-vector dense retrieval; often used as a re-ranker on top of hybrid results
- [[tf-idf]] — classical sparse retrieval method
- [[bm25]] — standard sparse retrieval baseline
- [[splade]] — learned sparse retrieval, state of the art

## Tags

#rag
