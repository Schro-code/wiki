# Wiki

Personal knowledge base covering AI/ML systems, infrastructure, and software engineering.

## Domains

| Domain | What's inside |
|---|---|
| `agentic-system-design/` | Design discipline for LLM-powered agent systems: patterns, failure modes, trade-offs |
| `agent-knowledge/` | Knowledge systems that learn passively — bitemporal graphs, ontology-driven extraction, reasoning |
| `rag/` | Retrieval-Augmented Generation — chunking, embeddings, hybrid search, BM25, ColBERT, SPLADE |
| `inference-engineering/` | Serving LLMs locally: model formats, quantization, inference servers, gateways, observability |
| `graphrag/` | Microsoft GraphRAG — pipeline, prompt tuning, query modes |
| `neo4j/` | Neo4j graph database — data model, Cypher, index-free adjacency |
| `system-design/` | Fundamentals: networking, protocols, distributed transactions, rate limiting, CAP/PACELC |
| `kubernetes/` | kubectl, kustomize, minikube |
| `harness/` | AI agent harness — the scaffolding layer that wraps LLMs into acting agents |
| `tts/` | Text-to-speech: mel spectrogram, acoustic model, vocoder, Kokoro |
| `scm/` | Software Configuration Management — versioning, branching, fork management, release engineering, supply chain |
| `devops/` | DevOps — culture, CI/CD pipelines, branching strategies, environments, deployment strategies |
| `testing/` | Software testing strategy — test pyramid, e2e, smoke, integration, and when to use each |
| `archify/` | Archify skill internals — two-stage pipeline, JSON schema contract, diagram type selection, explainer mode |
| `neuroscience/` | Neuroscience of learning and effort — effort conditioning (learned industriousness, peak-end), a discipline playbook with evidence map, and reward prediction error theory |
| `synthesis/` | Pages that integrate two or more domains |

## Structure

Each domain has an `index.md` that lists its pages and when to read each one. Start there before exploring individual files.

```
Wiki/
├── index.md          ← root index: all domains
├── tags.md           ← tag registry
├── log.md            ← operations log
└── <domain>/
    ├── index.md      ← domain index
    └── *.md          ← concept pages
```

## Not in this repo

- `labs/` — independent git repos with hands-on experiments
- `projects/` — private work project notes (meetings, decisions)
- `raw/` — raw source files ingested into wiki pages
