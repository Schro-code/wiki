---
domain: agent-knowledge
tags: [agent-knowledge]
sources: []
updated: 2026-10-14
---

# Utopia — Enterprise Knowledge System

## What it is

Utopia (by DeepLethe) is an open-source enterprise knowledge system. It ingests documents, extracts structured facts using an LLM, and stores everything in a bitemporal knowledge graph — a graph where every fact carries two timestamps: when it was true in the world, and when the system came to believe it.

A single Rust binary + one Postgres database. No external queues, no external search engines, no additional services.

---

## What problem it solves

Organizations accumulate knowledge from many sources — documents, databases, news, meetings. That knowledge **contradicts itself**, **changes over time**, and **nobody can trace why a decision was made**.

Most systems store the current truth and overwrite it when things change. When something goes wrong and someone asks *"what did we know, and when did we know it?"* — the answer is usually: *we don't know*.

Utopia's answer: make time, provenance, and conflict resolution **first-class**. Every fact knows where it came from, when it was believed, and whether it was ever contested. A knowledge base built on Utopia can answer not just *"what is true now?"* but *"what did we know on this date, and why did we believe it?"*

---

## How it solves it — the architecture

The system is built around two foundational ideas:

**1. Nothing is ever deleted.** Every fact is written as a new row. When a fact changes, the old version is closed (not erased), and the new version is written alongside it. This is what makes it possible to query any past state.

**2. Every fact has two time axes.** When the fact was true in the world, and when the system came to believe it. These are different things — and both are stored.

Everything else (conflict detection, reasoning, the audit trail) follows from these two principles.

---

## The storage concepts

Before the pipelines: the fundamental "boxes" where everything lives.

```
┌─────────────────────────────────────────────────────────────────┐
│  FILE STORE                                                      │
│  Raw document bytes saved on disk, referenced by SHA-256 hash.  │
│  Example: { hash: "a3f9...", bytes: <PDF content> }             │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  DOCUMENT REGISTRY                                               │
│  Metadata per document. Tracks where it is in the pipeline.     │
│  Example: { name: "contract.pdf", status: "extracting",         │
│             uploaded_by: "Laura", kb: "Legal KB" }              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  CHUNKS                                                          │
│  The document cut into overlapping pieces of text.              │
│  Each chunk also holds a vector: numbers representing its        │
│  meaning (computed by the embedding model).                     │
│  Example: { text: "Acme acquired DataCo in March 2024",         │
│             vector: [0.12, -0.44, 0.87, ...] }                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  SEARCH INDEX                                                    │
│  A keyword index over all chunk texts. Enables fast BM25         │
│  (keyword relevance) search. Embedded in the binary — no        │
│  external search engine.                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  ONTOLOGY  — the vocabulary                                      │
│  Two lists:                                                      │
│  - Entity types:   person, organization, product, event...      │
│  - Relation types: works_at, acquired, has_ceo, located_in...   │
│  Each relation also carries logical properties:                  │
│    functional     → only one valid value at a time              │
│    transitive     → A→B, B→C implies A→C                        │
│    symmetric      → A→B implies B→A                             │
│  Example: { key: "has_ceo", domain: organization,               │
│             range: person, functional: true }                    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  ENTITY REGISTRY  — graph nodes                                  │
│  Every real-world thing found in documents.                     │
│  Example: { name: "Acme Corp", type: organization,              │
│             aliases: ["Acme", "ACME Inc."] }                    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  FACT LEDGER  — graph edges, append-only                         │
│  Every assertion ever made. Nothing is ever deleted.            │
│  Example: { subject: Acme Corp,                                 │
│             predicate: has_ceo,                                 │
│             object: Laura Gómez,                                │
│             valid_from: 2020-03,    ← when true in the world    │
│             valid_to: 2023-11,                                  │
│             recorded_at: 2024-09,  ← when system learned it     │
│             invalidated_at: null } ← still believed             │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  EVIDENCE LINKS                                                  │
│  Bridge between Fact Ledger and Chunks.                         │
│  Every fact points to the exact text chunk it came from,        │
│  with the verbatim quote.                                       │
│  Example: { fact: "Acme acquired DataCo",                       │
│             chunk: paragraph 3 of contract.pdf,                 │
│             quote: "Acme Corp acquired DataCo in March 2024" }  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  JOB QUEUE                                                       │
│  Background work waiting to be done. Workers claim jobs         │
│  atomically (Postgres FOR UPDATE SKIP LOCKED).                  │
│  Example: { kind: "extract_document",                           │
│             payload: { document_id: "abc..." },                 │
│             status: "queued" }                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  CONFLICT QUEUE                                                  │
│  Facts that clash and need a human to decide.                   │
│  Example: { old: "Acme CEO = Laura (open)",                     │
│             new: "Acme CEO = Roberto",                          │
│             reason: "two values for a functional relation" }    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  AUDIT LEDGER  — immutable                                       │
│  Every human decision, forever. A database trigger blocks        │
│  UPDATE, DELETE, TRUNCATE — even from the application.          │
│  Example: { actor: "Laura", action: "closed old CEO fact",      │
│             at: 2024-09-08, detail: { kept: "Roberto" } }       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  DERIVED FACTS                                                   │
│  Facts the reasoning engine inferred automatically.             │
│  Stored separately from the Fact Ledger to prevent              │
│  accidental confusion with asserted ground truth.               │
│  Example: { subject: Acme Corp,                                 │
│             predicate: located_in,                              │
│             object: EU,                                         │
│             derived_from: [Acme→Madrid, Madrid→Spain,           │
│                            Spain→EU] }                          │
└─────────────────────────────────────────────────────────────────┘
```

### Grouped by role

```
KNOWLEDGE FOUNDATION    File Store, Document Registry, Chunks,
                        Search Index, Ontology, Entity Registry,
                        Fact Ledger, Evidence Links

WORKER                  Job Queue

GOVERNANCE              Conflict Queue, Audit Ledger

INFERENCE               Derived Facts
```

---

## The Ontology in depth

The Ontology is the vocabulary of a knowledge base. It is the only layer the user needs to understand and tune — everything else reads from it automatically.

### Where it comes from

Three sources, layered:

**1. Built-in packs** — chosen when creating a knowledge base. The binary ships with: `schema.org`, FOAF, PROV-O, W3C Org, IOF Core. They seed the initial entity and relation type rows.

**2. User editing** — via the ontology workbench in the UI. You write descriptions, mark relations as `functional` or `transitive`, set domain/range. Descriptions feed directly into the LLM extraction prompt — better descriptions produce better extractions.

**3. Ontology Agent (ADR 0061, rc9)** — replaced the old auto-extension heuristic. An LLM-driven agent reads unbound phrase signatures and kind words from the open graph, considers the current ontology and user-defined "competency questions," then proposes new entity types, relation types, properties, and implication rules — each with a definition, examples, and the questions it serves. Proposals require human approval (the ontology remains a contract per ADR 0012). Measured by two numbers: questions answered correctly and proposals people change. The old `Suggest` path, auto-extension, and adoption-by-wording were retired entirely.

### Relation properties and their effects

| Property | Meaning | Effect |
|---|---|---|
| `functional` | At most one value at a time | Triggers conflict detection when a second open fact arrives |
| `transitive` | A→B, B→C implies A→C | Reasoning engine derives the chain automatically |
| `symmetric` | A→B implies B→A | Reasoning engine derives the reverse |
| `asymmetric` | A→B implies NOT B→A | Violation detected as a conflict |

### Domain and range

Each relation can declare which entity types are allowed on each side:

```
works_at  domain: person      range: organization
acquired  domain: organization range: organization
```

If an extracted fact violates domain or range (e.g., a `place` as subject of `works_at`), it is dropped and logged — never silently.

---

## The two pipelines

### Pipeline 1 — Ingest (no LLM, runs fast)

Turns a file into searchable chunks.

```
Step 1 — STORE
  File bytes saved to FILE STORE under SHA-256 hash.
  Row created in DOCUMENT REGISTRY with status: queued.

Step 2 — PARSE  (CPU thread, no network)
  Format detected by magic bytes. Output: one clean text string.
  PDF → text extraction
  DOCX/XLSX/PPTX → XML parsing
  HTML → readability pass → Markdown
  CSV/TSV → direct
  Markdown → direct

Step 3 — CHUNK
  Text split into overlapping pieces at sentence/paragraph boundaries.
  Each piece → one row in CHUNKS.

Step 4 — INDEX  (keyword search, immediate)
  Every chunk written into SEARCH INDEX.
  BM25 keyword search is now live — no LLM needed.

Step 5 — EMBED  (optional, needs embedding model)
  Chunks sent to embedding model in batches of 16.
  Each chunk gets a vector stored alongside it.
  Enables semantic (ANN) search.
  If no model configured: document is still searchable by keyword only.

DOCUMENT REGISTRY ← status: ready
✓ The document is searchable from this point.
```

### Pipeline 2 — Extract (needs chat LLM, runs in background)

Turns chunks into graph facts.

```
Job enqueued: extract_document
Worker picks it up.

Step 6 — BUILD PROMPT  (per chunk)
  Read current ONTOLOGY from database.
  Format entity types + relation types as vocabulary list.
  Include entities seen in earlier chunks of this document.

  System prompt:
    "Entity types: person, organization, product...
     Relation types:
     - works_at (person → organization): employed by
     - acquired [event]: one company buys another
     - has_ceo (organization → person): functional"

  User prompt:
    "Already seen: k1 [organization]: Acme Corp
     Text: Acme Corp acquired DataCo in March 2024..."

Step 7 — CALL LLM
  Returns structured JSON:
    {
      "entities": [
        { "local_id": "e1", "name": "DataCo", "type": "organization",
          "specific_type": "data analytics company" }
      ],
      "facts": [
        { "subject": "Acme Corp", "subject_ref": "k1",
          "predicate": "acquired",
          "object": "DataCo", "object_ref": "e1",
          "valid_from": "2024-03",
          "confidence": 0.9,
          "quote": "Acme Corp acquired DataCo in March 2024" }
      ]
    }

  If rate-limited → exponential backoff + retry (up to 5 times).
  If response truncated → recover already-complete JSON objects.

Step 8 — VALIDATE each fact
  Is the entity name a real name? (not a sentence fragment)
    Checks: word count, presence of finite verbs, sentence structure.
    Fail → drop and log.
  Does the predicate exist in the ontology?
    No → record raw predicate on evidence, skip as a typed fact.
  Does domain/range match the entity types?
    Fail → drop and log.
  Is confidence ≥ 0.6?
    Fail → drop and log.
  All drops logged in extraction drops table — never silent.

Step 9 — ENTITY RESOLUTION
  For each entity the LLM found:
  1. Exact name + type match in ENTITY REGISTRY → reuse existing row.
  2. Embedding vector similar enough → merge candidate.
  3. Ambiguous → LLM adjudication: "same thing or two different things?"
     Uncertain answer → Review Queue (human decides).

Step 10 — WRITE TO FACT LEDGER
  Valid facts written as new rows (append-only, never overwrite).
  Each fact gets:
    valid_from  = date from the text (world axis)
    recorded_at = now               (knowledge axis)
  Evidence Link created: fact ↔ chunk + verbatim quote.

Step 11 — CONFLICT DETECTION
  If predicate is functional AND an open fact with same subject exists:
    existing: Acme Corp → has_ceo → Laura (valid_to: null)
    new:      Acme Corp → has_ceo → Roberto
  → Two open values for a "one at a time" relation = contradiction.
  → Row written to CONFLICT QUEUE. Human decides.

DOCUMENT REGISTRY ← status: extracted
✓ Facts are in the graph.
```

### The two phases are independent

```
Upload file
  │
  ▼
[INGEST]  ── no LLM ──► CHUNKS + SEARCH INDEX + vectors
  │                      ✓ searchable, chatttable
  │
  ▼ (new job enqueued)
[EXTRACT] ── LLM ──────► ENTITY REGISTRY + FACT LEDGER + EVIDENCE LINKS
                          ✓ in the knowledge graph
```

You can search and chat with a document while extraction is still running.

---

## How time works

### The two clocks on every fact

```
valid_from      when this became true in the real world
valid_to        when it stopped being true

recorded_at     when the system first believed this
invalidated_at  when the system stopped believing this
```

**World axis question:** "What was true about Acme Corp in 2022?"
→ filter: `valid_from ≤ 2022 AND (valid_to IS NULL OR valid_to ≥ 2022)`

**Knowledge axis question:** "What did we know about Acme Corp as of January 2024?"
→ filter: `recorded_at ≤ 2024-01 AND (invalidated_at IS NULL OR invalidated_at ≥ 2024-01)`

These are independent. A fact from a 2020 contract (`valid_from: 2020`) uploaded today (`recorded_at: 2024-09`) answers differently on each axis.

### The three states of valid_to

```
valid_to: null                  → still ongoing right now
valid_to: "2023-11"            → ended in November 2023
valid_to_precision: "unknown"  → ended, but source doesn't say when
                                  ("former CEO of Acme Corp")
```

Without the "unknown" state, "former CEO" would be stored as `valid_to: null` — which means *"still CEO"*. That's the opposite of what the source says.

### Date precision

The system never invents precision it doesn't have:

| Source text | Stored as | Precision |
|---|---|---|
| "in 2020" | 2020 | year |
| "in March 2024" | 2024-03 | month |
| "on 15 March 2024" | 2024-03-15 | day |

### When a fact changes

The old fact is never deleted. It is closed:

```
old row: valid_to = 2023-11, invalidated_at = (when we learned it changed)
new row: written fresh, supersedes = old row id
```

Both rows remain in the Fact Ledger, queryable on either time axis.

---

## Knowledge governance

### Conflict detection

When a new fact arrives for a `functional` relation and an open fact already exists:

```
existing: Acme Corp → has_ceo → Laura Gómez   (valid_to: null)
new:      Acme Corp → has_ceo → Roberto Sanz
```

The system writes a row to the **Conflict Queue** and surfaces it in the UI. The human has three choices:

| Decision | Meaning | What happens |
|---|---|---|
| Close the old | Laura left; Roberto is correct | Old fact gets `valid_to` |
| Keep both | Both are valid in different contexts | Both stay open |
| Reject the new | Source was wrong | New fact gets `invalidated_at` |

Every decision is written to the **Audit Ledger**.

### The Audit Ledger

Immutable by design. A database trigger blocks UPDATE, DELETE, and TRUNCATE — the only allowed operation is INSERT. Every entry stores:

- Actor name as a text snapshot (not just a foreign key — the user account might be deleted later, but the record must remain readable)
- Timestamp
- IP address and user agent
- The state of the object at decision time

### Reasoning engine

Derives new facts automatically from existing ones. Off by default. Runs periodically (default: every 60 minutes).

**Rules come from ontology axioms:**

```
transitive:  A → located_in → B, B → located_in → C
             ⟹  A → located_in → C

symmetric:   A → knows → B
             ⟹  B → knows → A
```

**Why derived facts are stored separately from the Fact Ledger:**

If derived facts lived in the Fact Ledger, every query would silently include inferences unless the developer remembered to filter. With a separate table, forgetting the JOIN means you simply don't see derivations — you cannot accidentally treat a machine inference as asserted ground truth.

**Lifecycle of a derived fact:**
```
created        when premises exist and the rule matches
invalidated    when any premise is removed (not deleted — closed)
confidence     = minimum confidence of all premises in the chain
```

**Example — why derivations matter:**
```
You assert:
  Acme Corp → headquartered_in → Madrid
  Madrid    → located_in       → Spain
  Spain     → located_in       → EU

Ontology: "located_in" is transitive.

Engine derives:
  Acme Corp → located_in → Spain  (derived)
  Acme Corp → located_in → EU     (derived)

Business rule: "companies located in EU require GDPR compliance"
→ automatically applies to Acme Corp.
   Nobody had to assert it manually.
```

---

## The agent query loop

When a user asks a question, the system doesn't run a single search and answer. It sets up an **agent loop**: the LLM receives a set of tools it can call, decides which ones to use, reads the results, and decides what to look for next — up to 6 rounds.

### The tools

| Tool | Reads from | Returns |
|---|---|---|
| `find_entities` | Entity Registry | Entities matching a name, with fact count |
| `entity_facts` | Fact Ledger + Derived Facts | All facts for one entity, at any point in time |
| `search_chunks` | Chunks (hybrid search) | Top 6 most relevant text passages |
| `get_document` | Chunks | Full text of a specific document |
| `changes` | Fact Ledger | What changed during a time window |
| `query_data` | Mounted external database | Read-only SQL results |
| `remember` | → Conflict Queue | Creates a pending fact (needs human approval) |

### Hybrid search (search_chunks)

Every `search_chunks` call runs two searches and fuses the results:

```
Query: "Acme Corp CEO 2024"
  │
  ├──► BM25 keyword search   → top 24 by term frequency
  │     via SEARCH INDEX
  │
  └──► ANN semantic search   → top 24 by vector similarity
        embed the query → compare against chunk vectors
  │
  RRF fusion → top 6 by combined rank → returned to LLM
```

RRF (Reciprocal Rank Fusion): a chunk's score is based on its rank in each list. A chunk ranked 2nd in both beats one ranked 1st in one list and absent in the other. Keyword and semantic results reinforce each other.

### Time-aware queries

```
entity_facts(entity_id, at: "2022-06-01")
  → world axis: facts valid on that date
  → knowledge axis: facts we had recorded by that date

entity_facts(entity_id, before: "2024-01-15")
  → knowledge axis only: what did we know just before that moment?
    (used to answer: "what did we believe then?")
```

The two axes are never collapsed into one parameter — that would conflate "what was true in March" with "what we knew in March".

### A complete example

**User asks:** *"Who is the CEO of Acme Corp, and when did they take the role?"*

```
Round 1 — LLM decides: "find Acme Corp in the graph"
  calls: find_entities(name: "Acme Corp")
  reads: ENTITY REGISTRY
  gets:  { id: "abc-123", type: organization, 47 facts }

Round 2 — LLM decides: "get its CEO facts"
  calls: entity_facts(entity_id: "abc-123")
  reads: FACT LEDGER
  gets:
    has_ceo → Laura Gómez    valid: 2020-03 → 2023-11
    has_ceo → Roberto Sanz   valid: 2024-01 → (open)

Round 3 — LLM decides: "confirm Roberto's start with a document"
  calls: search_chunks(query: "Acme Corp CEO Roberto Sanz 2024")
  reads: CHUNKS via BM25 + ANN, fused
  gets:  6 passages, including the announcement paragraph

LLM writes:
  "The current CEO of Acme Corp is Roberto Sanz, who took the role in
   January 2024 [1]. Previously, Laura Gómez served as CEO from March
   2020 to November 2023 [2]."

[1] → links to chunk → EVIDENCE LINKS → FACT LEDGER
[2] → links to chunk → EVIDENCE LINKS → FACT LEDGER
```

### The `remember` tool — why it requires approval

An agent can be told to remember a new fact via conversation:

```
User: "Remember that Acme moved its HQ to Shenzhen."
Agent calls: remember(subject: "Acme Corp", predicate: "located_in", object: "Shenzhen")
```

This does **not** write directly to the Fact Ledger. It creates a pending entry in the Review Queue. A human must confirm before the fact joins the graph. A conversational assertion from an agent should never silently modify ground truth.

### Full diagram

```
User question
      │
      ▼
  [LLM agent]
  ← last 20 turns of conversation
  ← up to 20 known entities from this session
  ← tool schema
      │
      │  up to 6 rounds of tool calls:
      │
      ├──► find_entities ────────► ENTITY REGISTRY
      ├──► entity_facts  ────────► FACT LEDGER  (world axis + knowledge axis)
      │                            DERIVED FACTS
      ├──► search_chunks ────────► CHUNKS
      │                              BM25 via SEARCH INDEX
      │                              ANN  via chunk vectors
      │                              RRF fusion → top 6
      ├──► get_document  ────────► CHUNKS (full document text)
      ├──► query_data    ────────► mounted external DB (read-only SQL)
      └──► remember      ────────► CONFLICT QUEUE (pending, needs approval)
                                         │
                                         ▼ (human approves)
                                   FACT LEDGER + AUDIT LEDGER
      │
      ▼
  Answer with citations [1][2][3] streamed to user
  Citations → CHUNKS → EVIDENCE LINKS → FACT LEDGER
```

---

## Zero configuration required

The system needs only two things configured before it works:

1. A **chat LLM endpoint** (OpenAI-compatible: OpenAI, DeepSeek, Ollama, vLLM...)
2. An **embedding endpoint** (same: any OpenAI-compatible embedder)

Upload a document and the system works. The ontology is pre-seeded from the pack chosen at knowledge base creation. The extraction prompt is built automatically from the current ontology. The vocabulary grows through the ontology agent, which proposes additions from the open graph for human approval.

Optional tuning (not required):
- Edit ontology descriptions → better extraction quality
- Mark a relation `functional` → enables conflict detection
- Mark a relation `transitive` → enables reasoning
- Turn on `materialize_inferences` → reasoning engine runs periodically

---

## rc9 sync — new capabilities (October 2026)

222 upstream commits merged. The major new systems and improvements:

### Ontology Agent (ADR 0061)

Replaces the old heuristic ontology growth loop entirely. The agent:
- Reads unbound phrase signatures and classless kind words from the open graph
- Proposes entity types, relation types, properties, and implication rules
- Each proposal carries a definition, examples, the signatures it would bind, and the competency questions it serves
- Requires human approval — the ontology remains a contract (ADR 0012)
- Approved elements carry regression cases that are rerun when definitions change
- Measured by two numbers: questions answered correctly, and proposals people edit/reject

The old `Suggest` path, auto-extension, and adoption-by-wording were retired (5,167 lines removed).

### Errata Agent

A post-extraction quality gate. For each document's typed facts:
- Receives structural flags (domain/range violations, absent names, non-date values in date fields)
- The LLM judges each fact as keep/retract/revise, must quote the document verbatim
- Retraction requires two votes and a flag; unflagged facts are never retracted by the agent alone
- Actions pass the impact gate before modifying the graph

### Business Rules with Joins (ADR 0047)

The reasoning engine now supports cross-entity rules:
- Rules can join two entities through a shared relation and conclude new relations
- Join evaluation is bucketed by subject (5.9s → 105ms for 100K pairs)
- Rule definitions are now versioned (ADR 0060): every edit opens a new version snapshot, and derived conclusions name the version they were drawn under

### Temporal Reasoning Overhaul (ADR 0064)

Fundamentally reworks how time expressions are handled during extraction:
- Time context is built per-chunk during extraction (not a single post-extraction call)
- Each statement is individually attested by the reference point in force at its quote's position
- Relative expressions ("three months later") resolve against the section's own date
- A fact with no time holds at every moment (null attestation); upload time no longer enters world time
- Deleted document attestation rebuilds from remaining evidence

Measured improvements: time expressions recorded rose from 18–20/29 to 27–29/29; heading-scoped dates went from 0/6 to 18/18.

### Name-Vector Recall

Entity recall now matches mentions by embedding vector, so abbreviations, transliterations, and alternate scripts find the same entity without exact string match. Names carrying different numbers are no longer paired ("Report Q1" vs "Report Q2" stay distinct).

### Phrase Alignment Improvements

- Staleness detection based on input fingerprint instead of timestamp (ADR 0053)
- Batches signatures by candidates, shortlists by embedding distance
- Inheritance-aware candidate properties
- Human phrase decisions are now asynchronous (ADR 0051): HTTP 202 with job ID
- Single-date statements bound to a state property now carry `marks` (start/end/none)

### Chat & UX

| Feature | Description |
|---|---|
| **Stop cancels generation** (ADR 0063) | Stop button cancels server-side generation and preserves the partial answer |
| **Copy answers & code blocks** | Each answer and code block has a copy button; works without Clipboard API |
| **Chat pagination** | Older conversations reachable through bounded pagination |
| **Failed answer retry** | Retry a failed question in place without storing a duplicate entry |
| **Long conversation handling** | Old exchanges are shed to stay within context window; context refusals are recovered |
| **Graph citations with sentences** | Graph answers cite the specific sentence each fact was read from |
| **Send question once** | Each question sent to the model once (was twice before) |
| **i18n** | Chat step fields, stream failure codes, and error messages localized |

### Parsers & Ingest

| Parser | Improvements |
|---|---|
| **PPTX** | Slide tables preserve rows/columns; merged cells read; line breaks keep words apart; cover date attests every slide |
| **DOCX** | Headings parsed as headings (per-section chunking/dating); text boxes read once; non-breaking hyphens handled; paragraph-to-Markdown block breaks fixed |
| **Spreadsheets** | Dates read as displayed date (not Excel day count); 15-digit precision; merged cells filled; capped content warns what was left out |
| **PDF** | Characters use actual glyphs, not look-alike radicals/ligatures |
| **OCR** (ADR 0065) | Provider choice: MinerU (default) or Volcengine Ark vision model, configurable per workspace |
| **Notion** | Nested blocks read recursively; page bodies preserved on failures; table cells imported |
| **CSV** | Empty fields preserved; delimited records keep their own row |
| **Chunks** | Named by section; date detection reads filename first; Markdown code blocks excluded from time scoping |

### Data Sources & Integrations

| Source | What's new |
|---|---|
| **ClickHouse** | Read-only warehouse mounting with server timeout and KILL QUERY |
| **Databricks** | Follows result chunk links to retain all rows |
| **Snowflake** | Reads result partitions by statement handle |
| **WebDAV** | URL escapes preserved |
| **GitHub** | Preserves unchanged issue comments on incremental sync |
| **Source Push** (ADR 0054) | `POST /sources/{id}/statements` — structured data enters as open statements without LLM round-trip |

### API & Contracts

| Endpoint / Feature | Description |
|---|---|
| **Document Content API** (ADR 0052) | `GET /documents/{id}/content` serves retained original bytes; `GET /documents/{id}/versions` returns the version ledger |
| **OIDC Token Exchange** (ADR 0066) | `POST /auth/oidc/exchange` — apps sharing the same IdP can swap a fresh ID token for a Utopia session |
| **RDF Export: Contested Facts** (ADR 0062) | Export includes `utopia:FactConflict` and `utopia:AxiomViolation` resources |
| **SHACL Import** | A SHACL shape gives a property the domain and range it does not declare |
| **Expression Validation** (ADR 0049) | Rule expressions validated for unit compatibility at write time |
| **Action Dispatch** (ADR 0050) | Durable at-most-once execution model for external action dispatch |

### Infrastructure

- **Multi-arch Docker images** — linux/amd64 + linux/arm64
- **CI overhaul** — lint, test, build as parallel jobs; all tests run against real Postgres
- **Performance** — ontology views optimized (LEFT JOIN + concurrent fetch); graph rebuilds sped up with FK indexes; embedding reused per chat turn; governance serialized (no 10x concurrent duplication)
- **Security** — login response time equalized (timing attack fix); PAT expiration validated before issuance; vector index built by table-owning role
- **Workspace reasoning effort** — every model call respects the workspace setting; alignment no longer thinks on its own

### Architecture Decision Records (ADRs 0049–0066)

| ADR | Title |
|---|---|
| 0049 | Expression declarations are checked when a rule is written |
| 0050 | An action attempt keeps its identity and uncertain outcome |
| 0051 | A human phrase decision carries its materialization work |
| 0052 | Document content is a read contract over the retained ledger |
| 0053 | A phrase decision records the inputs it considered |
| 0054 | A source may push statements in the open contract |
| 0060 | A rule's definition has a history |
| 0061 | The ontology is proposed from the open graph and judged by its questions |
| 0062 | The export says what is contested |
| 0063 | Stopping a chat ends the generation |
| 0064 | Extraction reads time as it goes |
| 0065 | The OCR reader chooses its provider |
| 0066 | Another app may trade a fresh ID token for a session |

---

## Relations

- [[rag/hybrid-search]] — the BM25 + ANN + RRF pattern used in search_chunks
- [[graphrag/index]] — GraphRAG takes a similar approach to knowledge graph construction but with a different architecture
- [[harness/index]] — the agent loop pattern is related to harness tool-calling loops

## Tags

#agent-knowledge
