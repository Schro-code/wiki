---
domain: rag
type: index
updated: 2026-09-07
---

# RAG — Index

Retrieval-Augmented Generation: chunking, embedding, vector databases, retrieval metrics, hybrid and filtered search.

## Pages

| Page | When to look | Sources |
|---|---|---|
| [[rag-fundamentals]] | Look when you want to understand the full RAG pipeline — chunking, metadata, embeddings, metrics, vector database comparison, hybrid search, filtered search, ColBERT | labs/lda-vectordb-benchmark |
| [[colbert]] | Look when you want to understand ColBERT — why pooled embeddings fail on negation, how MaxSim works, two-stage retrieval vs re-ranking, when to use it | labs/lda-vectordb-benchmark |
| [[hybrid-search]] | Look when you want to understand the full hybrid search landscape — dense vs sparse, TF-IDF / BM25 / SPLADE, RRF and other fusion methods, when to use each | labs/lda-vectordb-benchmark |
| [[bm25]] | Look when you want to understand BM25 — term frequency with saturation, IDF, length normalization, why it breaks on Spanish morphology | labs/lda-vectordb-benchmark |
| [[tf-idf]] | Look when you want to understand TF-IDF — the formula, sparse vectors, dynamic corpus problem, when to use over BM25 | labs/lda-vectordb-benchmark |
| [[splade]] | Look when you want to understand SPLADE — learned sparse retrieval, query expansion, sparsity regularization, vs BM25 and dense embeddings | labs/lda-vectordb-benchmark |
