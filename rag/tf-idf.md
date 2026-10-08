---
domain: rag
tags: [rag]
sources: []
updated: 2026-09-07
---

# TF-IDF — Term Frequency / Inverse Document Frequency

## What it is

A classical sparse retrieval scoring function that multiplies how often a term appears in a document (TF) by how rare that term is across the entire corpus (IDF). The predecessor of BM25 and the foundation of most keyword-based retrieval systems.

## What problem it solves

Given a query and a corpus, rank documents by relevance using only term statistics — no embeddings, no parameters, no training.

## Notes

### The formula

```
TF(term, doc)  = count of term in document
IDF(term)      = log(N / df)
TF-IDF(term, doc) = TF × IDF
```

For a multi-word query, sum across all terms:
```
score(query, doc) = Σ TF-IDF(term, doc)   for each term in query
```

Where:
- `N` = total documents in the corpus
- `df` = number of documents containing the term

**IDF intuition:** a term appearing in every document (like "el") has IDF ≈ 0 — it adds nothing. A term appearing in 2 out of 382 documents has IDF = log(382/2) ≈ 5.25 — highly discriminative.

**Example with 382 chunks:**

| Term | df | IDF | Signal |
|---|---|---|---|
| "el" | 380 | log(382/380) ≈ 0.005 | Near zero — useless |
| "seguro" | 200 | log(382/200) ≈ 0.65 | Low — too common |
| "lunas" | 15 | log(382/15) ≈ 3.24 | High — domain-specific |
| "panorámicas" | 2 | log(382/2) ≈ 5.25 | Very high — very rare |

---

### TF-IDF as sparse vectors

Classical TF-IDF produces a scalar score per (query, document) pair at query time. The modern use in hybrid search precomputes **sparse vectors** at index time instead.

Each document becomes a vector where each dimension corresponds to a vocabulary term, and the value is the TF-IDF score for that term:

```
vocabulary: [el=0, seguro=1, cubre=2, lunas=3, panorámicas=4, ...]

document "El seguro cubre lunas panorámicas":
sparse vector: [0.005, 0.65, 2.1, 3.24, 5.25, 0, 0, 0, ...]
                                                  ↑ terms not in doc = 0
```

The vector has 30,000+ dimensions (full vocabulary size) but is mostly zeros — hence "sparse." At query time, the query is embedded the same way and similarity is a dot product between two sparse vectors. Only non-zero dimensions contribute, so it's fast despite the high dimensionality.

**Why this avoids the stemming problem:** the vocabulary is built from the actual corpus. Terms like `cobertura` and `cubierto` get their own positions based on observed co-occurrence — no stemmer needed. The corpus itself carries the language signal.

---

### TF-IDF vs BM25 — what BM25 fixes

| Problem with raw TF | BM25 fix | TF-IDF |
|---|---|---|
| Long documents always win (more words = more occurrences) | Length normalization (b parameter) | ✗ Not fixed |
| 50th occurrence as valuable as 1st | Term saturation (k1 parameter) | ✗ Not fixed |
| Edge cases (df=0, N=df) | Smoothed IDF formula (+0.5) | ✗ Basic log(N/df) |

TF-IDF is simpler but less robust. For most production use cases BM25 is preferred over raw TF-IDF as a sparse baseline.

---

### The dynamic corpus problem

IDF depends on the full corpus at index time:

```
IDF(term) = log(N / df)
```

When new documents arrive, both `N` and `df` change — which means IDF scores for all existing terms change, and all existing document vectors become stale. Three strategies:

| Strategy | How | Trade-off |
|---|---|---|
| Periodic recomputation | Recompute all vectors nightly/weekly | Stale between runs, expensive at scale |
| Incremental approximation | Update IDF as docs arrive | Introduces drift, complex to implement |
| Frozen vocabulary | Compute once on a large snapshot, freeze | IDF drift is small if snapshot is representative — most common in production |

SPLADE avoids this problem entirely — its sparse vectors come from a neural model that doesn't depend on corpus statistics. Adding a document requires no recomputation.

---

### When to use TF-IDF sparse vectors

- Corpus is relatively static (policy documents, reference manuals)
- You need corpus-trained sparse retrieval without a neural model
- Language is non-English (avoids BM25's English stemmer dependency)
- Budget/infrastructure constraints prevent SPLADE deployment

Avoid when the corpus grows continuously — use SPLADE or BM25 with an English stemmer swap instead.

## Relations

- [[hybrid-search]] — TF-IDF sparse vectors are one of the sparse retrieval options
- [[bm25]] — BM25 adds saturation and length normalization on top of TF-IDF
- [[splade]] — learned sparse alternative that avoids the dynamic corpus problem and adds query expansion

## Tags

#rag
