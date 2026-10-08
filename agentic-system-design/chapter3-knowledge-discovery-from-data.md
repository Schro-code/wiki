---
domain: agentic-system-design
tags: [agentic-system-design, rag]
sources: [ai-agent-book/chapter3.md]
updated: 2026-09-21
---

# Knowledge Discovery from Structured Data

## What it is

A two-phase pipeline that extracts implicit, tacit knowledge buried in large structured datasets (case records, transaction logs, medical records) and converts it into explicit decision logic an agent can apply. It is the leap from **information retrieval** (find matching records) to **knowledge discovery** (mine patterns that explain outcomes).

## What problem it solves

Not all knowledge lives in documents. In legal, medical, and financial domains, the knowledge that shapes decisions is distributed across thousands of case outcomes — how judges weight conflicting factors, how doctors adjust treatment plans, how underwriters assess risk. This knowledge is never written down as rules; it emerges from statistical patterns. Standard RAG, which retrieves document fragments, cannot surface it. You need to *analyze the data* to distill it.

---

## Notes

### The two phases

**Phase 1 — Knowledge Extraction and Structuring**

Convert unstructured case descriptions into standardized JSON objects containing all key judgment factors.

- Use an LLM to read raw cases and extract structured fields
- Use a **bottom-up schema discovery** strategy: have the LLM analyze hundreds of sample cases and freely enumerate the factors it observes, rather than imposing a rigid schema upfront
- This produces a schema that fits the data, not one that fits the designer's prior assumptions

Schema design pattern — modular, not monolithic:

| Schema layer | What it covers |
|---|---|
| **Core schema** | Factors common to all cases: voluntary_surrender, compensation_amount, prior_convictions |
| **Charge-specific extension** | Factors relevant only to a crime type: theft → amount_involved; negligent injury → injury_severity_grade |

The modular design avoids polluting all cases with irrelevant fields.

---

**Phase 2 — Factor Analysis and Importance Modeling**

Translate the structured data into numerical feature vectors, then apply statistical/ML techniques to discover patterns.

**Encoding choices:**

| Field type | Encoding | Why |
|---|---|---|
| Categorical (crime type: theft / robbery / fraud) | One-hot vector `[1,0,0]` | Avoids implying ordinal magnitude — "fraud=3" would mislead most algorithms into treating it as "more severe" |
| Binary (voluntary_surrender: yes/no) | `1` / `0` | Direct |
| Continuous (compensation amount) | Log transformation + standardization | Reduces skew from large outlier values |

**Clustering → case prototypes:**

After vectorization, apply clustering (e.g., KMeans) to discover natural groups of cases. Each cluster is a **case prototype** — a representative pattern such as "unarmed brawl triggered by minor quarrel, victim slightly injured" or "premeditated armed gang assault, victim seriously injured."

**Factor importance hierarchy:**

Analyze the defining features of each cluster to quantify which factors most strongly differentiate outcomes. This produces a ranked importance model: the agent learns which factors to ask about first and how to weight them in its analysis.

---

### How this knowledge becomes agent behavior

The Factor Importance Hierarchy Model drives the agent's conversational data collection:

1. **Guide questions by factor importance** — ask the user about the most impactful factors first, not in arbitrary order
2. **Retrieve similar case prototypes** — once enough factors are known, find the closest cluster in the knowledge base
3. **Data-driven analysis** — present the prototype's statistical distribution (typical sentencing ranges, common outcomes) as evidence-backed reasoning

This is distinct from retrieval-only RAG: the agent doesn't retrieve facts to answer a question — it applies learned decision logic to evaluate a new situation.

---

### Avoiding the black box

The "Factor Importance Hierarchy" exists precisely to avoid a naive black-box approach (feeding case features into a neural network and predicting an outcome directly). The issue with end-to-end prediction: it gives an answer but cannot explain why.

The pipeline here is interpretable by design:
- Clusters are human-readable case prototypes, not latent embeddings
- Factor importance is an explicit ranking, not a gradient attribution
- The agent presents the prototype + its statistics, not a point prediction

Interpretability matters whenever the agent must justify its reasoning to a human expert (lawyer, doctor, underwriter).

---

### Generalizing the pattern

The same two-phase structure applies beyond legal:

| Domain | Structured data source | Knowledge discovered |
|---|---|---|
| Legal | Court judgment records | Sentencing factor weights, recidivism patterns |
| Medical | Clinical records | Treatment outcome patterns by patient profile |
| Insurance | Claims history | Risk factor importance by policy/claim type |
| Finance | Transaction logs | Fraud indicator patterns, credit risk clusters |

Wherever expert intuition is encoded implicitly in historical outcomes, this pipeline can extract it.

---

### Key engineering decisions

- **Bottom-up schema discovery over top-down** — let the data reveal its own structure before locking the schema. Iterate the schema as more cases are analyzed.
- **Encode with domain awareness** — one-hot for nominal categories is rarely obvious; document why for each field to avoid regression if schema evolves.
- **Cluster count is a hyperparameter** — too few clusters and the prototypes are too coarse; too many and each cluster has too few cases to be statistically meaningful. Tune against your domain's granularity needs.
- **This is offline, not online** — the extraction and clustering happen in a preprocessing pipeline, not at query time. The agent loads the learned model, not the raw data.

---

## Relations

- [[chapter3-agentic-rag]] — retrieval orchestration used during the application phase once the knowledge base is built
- [[chapter3-knowledge-organization]] — RAPTOR and GraphRAG for unstructured knowledge; this page handles structured dataset knowledge
- [[chapter3-knowledge-updates]] — pipeline design for keeping the extracted knowledge base current as new cases arrive
- [[chapter3-user-memory-systems]] — for contrast: this page extracts patterns from many cases; user memory extracts facts about one user

## Tags

#agentic-system-design #rag
