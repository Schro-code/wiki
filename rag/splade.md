---
domain: rag
tags: [rag]
sources: []
updated: 2026-09-07
---

# SPLADE — Sparse Lexical and Expansion Model

## What it is

A learned sparse retrieval model that combines the speed of inverted-index keyword search with partial semantic understanding. Produces sparse vectors like TF-IDF but learns term weights and query expansion from (query, relevant document) training pairs.

## What problem it solves

TF-IDF and BM25 fail when a query and a relevant document use different words for the same concept:

> Query: *"¿Qué pasa si me roban el coche?"*
> Relevant chunk: *"Cobertura por sustracción del vehículo..."*

Neither "roban" nor "coche" appears in the chunk. TF-IDF and BM25 score this pair near zero. Dense embeddings solve this but lose the speed and interpretability of sparse retrieval.

SPLADE solves it by **expanding the query with related terms** — keeping the sparse format while capturing semantic relationships.

## Notes

### The core idea — query expansion

SPLADE doesn't replace "roban" with "sustracción". It keeps "roban" and adds "sustracción", "robo", "vehículo", "hurto" as additional weighted terms:

```
input query:  "roban el coche"

TF-IDF sparse: [roban=1.2, coche=0.8, 0, 0, 0, ...]

SPLADE sparse: [roban=1.2, coche=0.8, sustracción=0.6,
                vehículo=0.5, robo=0.9, hurto=0.4, 0, 0, ...]
```

The expanded terms come from what the model learned during training — that "roban" co-occurs with "sustracción" and "robo" in relevant document pairs across millions of examples.

The output is still a sparse vector — compatible with standard inverted indexes, fast at query time.

---

### Architecture

SPLADE uses two BERT-based encoders trained jointly:

- **Query encoder** — takes the query, produces a sparse vector with expanded terms
- **Document encoder** — takes the document, produces a sparse vector upweighting retrieval-relevant terms

Both encoders output weights over the full vocabulary (~30,000 dimensions). Similarity is a dot product between query and document sparse vectors — only non-zero dimensions contribute.

---

### Training

Training data: **(query, relevant document) pairs** — the same format used to generate ground truth in the insurance benchmark.

The loss function has two components:

```
loss = retrieval_loss + λ × sparsity_penalty
```

**Retrieval loss** — punishes low dot product between relevant (query, document) pairs, rewards high dot product. This is where the model learns which term expansions actually help retrieval.

**Sparsity penalty** — penalizes every non-zero dimension:

```
sparsity_penalty = Σ |weight|   across all vocabulary dimensions
```

Without this penalty, BERT would produce non-zero values for all 30,000 vocabulary terms — effectively a dense vector. The penalty forces the model to only activate dimensions that justify their cost.

The model faces a trade-off on every training step:
- Activate more terms → better retrieval score → higher sparsity penalty
- Activate fewer terms → lower penalty → worse retrieval score

The result: vectors with ~20-200 non-zero dimensions out of 30,000. Genuinely sparse, compatible with inverted indexes, but semantically richer than BM25.

`λ` is a hyperparameter — higher λ = more aggressive sparsity = faster retrieval, less semantic coverage.

---

### Why it avoids the dynamic corpus problem

TF-IDF and BM25 depend on corpus statistics (N, df) computed at index time. When new documents arrive, IDF scores change and all existing vectors become stale.

SPLADE vectors are produced by a neural model that takes a document as input and produces a sparse vector — no corpus statistics involved. Adding a new document requires only one forward pass through the document encoder. No recomputation of existing vectors.

---

### SPLADE vs TF-IDF vs BM25

| | TF-IDF | BM25 | SPLADE |
|---|---|---|---|
| Term weights | Frequency-based | Frequency + saturation + length norm | Learned from training data |
| Rare term handling | High IDF | High IDF | Learned importance |
| Query expansion | No | No | Yes — learned |
| Stemming needed | No | Yes (English-tuned) | No |
| Dynamic corpus | Recompute IDF | Recompute IDF | One forward pass per doc |
| Training required | No | No | Yes — (query, doc) pairs |
| Interpretable | Yes | Yes | Yes — sparse weights visible |
| Speed | Fast | Fast | Fast (inverted index) |
| Semantic understanding | None | None | Partial |

SPLADE occupies the space between sparse and dense: sparse speed and interpretability, with partial semantic understanding.

---

### SPLADE vs dense embeddings

| | Dense (E5, BGE) | SPLADE |
|---|---|---|
| Semantic understanding | Full | Partial (term-level) |
| Interpretable | No | Yes — which terms activated |
| Index type | Vector index (ANN) | Inverted index |
| Query expansion | Implicit (vector space) | Explicit (term weights) |
| Dynamic corpus | One embedding per doc | One forward pass per doc |
| Multilingual | Yes (if trained on multilingual data) | Yes (if trained on multilingual data) |

In hybrid search, SPLADE replaces BM25/TF-IDF as the sparse component. Dense embeddings still handle the semantic side. The two complement each other.

---

### When to use SPLADE

- Corpus is multilingual or domain-specific (insurance, legal, medical)
- You need sparse retrieval without stemmer dependencies
- Corpus grows continuously (no IDF recomputation)
- You want interpretable term weights alongside semantic matching

Avoid SPLADE when:
- No labeled (query, document) pairs exist for your domain — training quality depends on them
- Infrastructure cannot run a neural model at index time
- English-only corpus with existing BM25 pipeline — the gain over BM25+Spanish stemmer may not justify the complexity

---

### In the benchmark context

The insurance benchmark used corpus-trained TF-IDF sparse vectors (not SPLADE) for the sparse component. SPLADE would be the next step — particularly for negation and cross-category queries where the term expansion could bridge "roban" → "sustracción" and "no cubre" → "excluido", "queda fuera".

Qdrant supports SPLADE natively via sparse named vectors. Azure AI Search requires external SPLADE inference with results stored as a custom field.

## Relations

- [[hybrid-search]] — SPLADE is the state-of-the-art sparse retrieval option in the hybrid search landscape
- [[tf-idf]] — the classical sparse baseline SPLADE replaces
- [[bm25]] — the standard sparse baseline SPLADE outperforms on multilingual and domain-specific text
- [[colbert]] — both are learned retrieval models; ColBERT is dense multi-vector, SPLADE is sparse single-vector
- [[rag-fundamentals]] — context: the insurance benchmark used TF-IDF sparse; SPLADE is the natural upgrade

## Tags

#rag
