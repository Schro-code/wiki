---
domain: graphrag
tags: [graphrag]
sources:
  - raw/GraphRAG The Marriage of Knowledge Graphs and RAG Emil Eifrem.md
  - raw/GraphRAG Getting Started.md
  - raw/GraphRAG Auto Tuning.md
  - raw/GraphRAG Manual Tuning.md
updated: 2026-08-31
---

# Microsoft GraphRAG

## What it is

The complete Microsoft GraphRAG system: the offline pipeline that transforms raw text into a knowledge graph, the optional prompt customization steps that tune extraction quality, and the three query modes that retrieve answers from the graph.

## What problem it solves

Unstructured text has no graph structure. Before GraphRAG can traverse a graph, the graph must be built from source documents via LLM extraction. Once built, different question types need different retrieval strategies — targeted entity questions, broad thematic questions, and exploratory connection questions each work better with a different approach.

## Notes

**System overview**

```
OFFLINE (build)                    ONLINE (query)
──────────────────────────────     ──────────────────────────────────────
[prompt-tune]  ← optional
    ↓
1. chunking                        question
2. LLM extraction (per chunk)          ↓
3. merge + deduplication       →   local search   (entities + neighborhood)
4. community detection         →   global search  (community summaries, MapReduce)
5. community summaries (LLM)   →   drift search   (iterative graph traversal)
    ↓
Parquet files
```

---

## Preprocessing — the 5 stages

Runs once with `graphrag index`. Expensive (many LLM calls). Produces the knowledge graph stored in Parquet files.

---

**Stage 1 — Chunking**

Fixed-size sliding windows (default: ~300 tokens, ~100 overlap). Overlap ensures entities near a boundary are captured in at least one full chunk. This is where the text loses its original structure.

---

**Stage 2 — LLM Extraction**

For every chunk, the LLM extracts entities (nodes) and relationships (edges).

**Entities:**
```
Entity: SCROOGE
Type:   person
Desc:   A cold, miserly businessman who despises Christmas and charity
```

**Relationships:**
```
SCROOGE ──→ MARLEY         "Former business partners; Marley's ghost visits Scrooge"
SCROOGE ──→ GHOST          "The ghost appears to Scrooge to warn him of his fate"
MARLEY  ──→ GHOSTLY CHAIN  "The chain Marley carries, forged by his greed in life"
```

The LLM only sees one chunk at a time — no global context. The same entity can appear under different names across chunks ("Scrooge", "Mr. Scrooge", "Ebenezer").

---

**Stage 3 — Merge and Deduplication**

All per-chunk extractions are merged into a single graph:
- Exact name matching
- Embedding similarity (catches "Scrooge" vs "Mr. Scrooge")

Edge weight = how many chunks two entities appeared together in. After merge: one canonical node per entity, edges aggregated and weighted by co-occurrence frequency.

---

**Stage 4 — Community Detection (Leiden algorithm)**

Leiden partitions the graph into communities: clusters of nodes more connected to each other than to the rest.

*Step 0 — every entity starts as its own community:*
```
[SCROOGE] [MARLEY] [GHOST] [FOG] [FROST] [CHARITY] [GENTLEMEN] ...
```

*Step 1 — each node asks: "am I better off joining a neighbor?"*
```
SCROOGE sees: edge to MARLEY (weight 3), edge to GHOST (weight 2), edge to FOG (weight 0)
→ pulls toward MARLEY and GHOST, ignores FOG
```

*Step 2 — merged communities become super-nodes, repeat until no improvement:*
```
Round 1: 40 nodes  → 15 communities
Round 2: 15 nodes  → 7 communities
Round 3: 7 nodes   → 5 communities  ← stop
```

**Final communities from A Christmas Carol (1/4 of the book):**

| Community | Entities | What it represents |
|---|---|---|
| Supernatural event | HOUSE, BELL | The bell that rings on its own — first sign of Marley's ghost |
| Charitable fundraising | CHARITY, GENTLEMEN | The scene where Scrooge refuses to donate |
| Winter setting | CITY, FOG, FROST, HOUSE GATEWAY | The cold gothic atmosphere outside Scrooge's house |
| Main characters | SCROOGE, MARLEY, GHOST, FRED, JACOB | Core characters and their relationships |
| Symbolism | MARLEY, GENIUS OF THE WEATHER, GHOSTLY CHAIN | Marley's chain = guilt; weather = mourning |

Key properties: non-deterministic (good enough in practice), guaranteed-connected (every node can reach every other within its community), hierarchical (level 0 = finest granularity, higher levels = broader themes).

---

**Stage 5 — Community Summaries**

The LLM writes a natural-language summary for each community based on its entities and relationships. Stored in `output/community_reports.parquet`. This is what global search queries at runtime — it never reads raw text again.

---

**Output files (all Parquet):**

| File | Contents |
|---|---|
| `entities.parquet` | id, title, type, description, degree |
| `relationships.parquet` | source, target, description, weight |
| `communities.parquet` | id, level, entity_ids |
| `community_reports.parquet` | title, summary, findings, rating |
| `text_units.parquet` | original chunks + which entities they contain |

```python
import pandas as pd
pd.read_parquet("output/entities.parquet")[["title","type","description"]].head(20)
```

---

## Prompt customization

The extraction and query prompts can be customized to improve results for specific domains. Two approaches: auto-tuning (LLM-generated) and manual (hand-edited). Workflow: auto-tune first, then refine manually.

---

### Auto-tuning (`graphrag prompt-tune`)

Optional pre-indexing step. Samples your documents, uses an LLM to understand what entities and relationships matter in your domain, and generates domain-specific versions of the three extraction prompts.

The default prompts extract "people, places, organizations, and concepts" — useful for general text but blind to domain structure. A legal corpus needs "statutes, clauses, parties, jurisdictions". A medical corpus needs "symptoms, diagnoses, treatments, drugs".

**What it rewrites:**

| Prompt file | Used at stage |
|---|---|
| `prompts/extract_graph.txt` | Stage 2 — entity/relationship extraction |
| `prompts/summarize_descriptions.txt` | Stage 3 — entity description merging |
| `prompts/community_report.txt` | Stage 5 — community summarization |

**How it works internally:**
1. Loads input documents and splits into small chunks (default: 200 tokens)
2. Selects a representative sample (default: 15 chunks)
3. Infers the domain — or uses the domain you pass with `--domain`
4. Runs LLM calls against the sample to learn what entity types appear
5. Writes tuned prompts to the output folder (default: `prompts/`)

**Document selection methods:**

| Method | Behavior | When to use |
|---|---|---|
| `random` (default) | Picks N chunks at random | Most cases |
| `top` | First N chunks | If docs have a general intro at the start |
| `all` | Every chunk | Small corpora only (expensive) |
| `auto` | Nearest K chunks to corpus centroid | Large, diverse datasets |

**The `--discover-entity-types` flag:**
- `--no-discover-entity-types` (recommended): entity types fixed to person, org, location, event — faster and more consistent
- `--discover-entity-types`: LLM invents entity categories from the text — flexible but less predictable; use when you don't know what entity types to expect

**CLI:**
```bash
# minimal — recommended starting point
graphrag prompt-tune --root /path/to/project --no-discover-entity-types

# explicit domain — faster, skips inference step
graphrag prompt-tune --root /path/to/project --domain "victorian literature" --no-discover-entity-types

# auto selection — better for large diverse datasets
graphrag prompt-tune --root /path/to/project --selection-method auto --no-discover-entity-types
```

**After running, update `settings.yaml`** (without this, `graphrag index` still uses the default templates):
```yaml
extract_graph:
  prompt: "prompts/extract_graph.txt"

summarize_descriptions:
  prompt: "prompts/summarize_descriptions.txt"

community_reports:
  prompt: "prompts/community_report.txt"
```

---

### Manual tuning

Direct editing of the prompt files. Every prompt is a plain text file with `{token_name}` placeholders that GraphRAG substitutes at runtime. Rewrite the instructions, tone, and examples around those placeholders — the tokens themselves must stay intact.

```
You are extracting entities from the following text:

{input_text}          ← GraphRAG injects the chunk here

Extract all entities of type: {entity_types}
```

Removing a required token breaks the pipeline stage that depends on it.

**Two categories:**

*Indexing prompts* — control what goes into the graph. Changes require a full re-index:

| Prompt | Stage | What it controls |
|---|---|---|
| `extract_graph.txt` | 2 | What entities and relationships the LLM extracts per chunk |
| `summarize_descriptions.txt` | 3 | How multiple descriptions of the same entity are merged |
| `extract_claims.txt` | optional | What facts/claims to extract (disabled by default) |
| `community_report.txt` | 5 | How the LLM writes each community summary |

*Query prompts* — control how the graph answers questions. Changes take effect immediately:

| Prompt | Search mode | What it controls |
|---|---|---|
| `local_search_system_prompt.txt` | local | Instructions and format for local search answers |
| `global_search_map_system_prompt.txt` | global | How each community summary is processed (map step) |
| `global_search_reduce_system_prompt.txt` | global | How partial answers are merged (reduce step) |
| `global_search_knowledge_system_prompt.txt` | global | Use of general knowledge from model training |
| `drift_search_system_prompt.txt` | drift | Instructions for drift (exploratory) search |

**Token reference — indexing prompts:**

*extract_graph:* `{input_text}`, `{entity_types}`, `{tuple_delimiter}`, `{record_delimiter}`, `{completion_delimiter}`

*summarize_descriptions:* `{entity_name}`, `{description_list}`

*extract_claims:* `{input_text}`, `{tuple_delimiter}`, `{record_delimiter}`, `{completion_delimiter}`, `{entity_specs}`, `{claim_description}`

*community_report:* `{input_text}` — tables of entities and relationships for the community

**Token reference — query prompts:**

All search prompts: `{response_type}`, `{context_data}`

Drift additionally: `{community_reports}`, `{query}`

**Workflow:**
```
graphrag prompt-tune      ← auto-generates domain-specific starting points
    ↓
graphrag index            ← run the index, inspect entities.parquet and community_reports.parquet
    ↓
graphrag query            ← ask questions, observe the answers
    ↓
edit prompt files         ← fix what auto-tuning couldn't guess
    ↓
graphrag index (again)    ← only if indexing prompts changed
```

**Common manual fixes:**
- Extraction misses an entity category → add examples to `extract_graph.txt`
- Community summaries are too generic → tighten `community_report.txt`
- Query answers are the wrong length or format → adjust `{response_type}` in the search prompts
- Too many low-value entities extracted → restrict `{entity_types}` or add a "do not extract" instruction

---

## Querying — the 3 modes

Runs on demand with `graphrag query`. Reads the Parquet files built during preprocessing.

---

### Local search

**Use for:** questions about specific entities, relationships, or events.

1. Query is embedded and matched against entity descriptions (vector search)
2. Most relevant entities become **anchor nodes**
3. GraphRAG traverses from anchors: neighbors, relationships, relevant text chunks
4. All retrieved context is packed into the LLM prompt

Fast and precise. Traversal is limited to the anchor's neighborhood.

**Examples:** *"Who is Ebenezer Scrooge and what are his main relationships?"* · *"What happened between Scrooge and Marley?"*

```bash
graphrag query --root ./ragtest --method local --query "Who is Scrooge?"
```

Prompt: `local_search_system_prompt.txt`

---

### Global search

**Use for:** thematic, comparative, or corpus-wide questions that no single chunk can answer.

1. Loads `community_reports.parquet`
2. Sends the question to each community summary in parallel (map)
3. Each call returns a partial answer
4. One final LLM call merges all partial answers (reduce)

Never reads original documents at query time — only the pre-computed summaries. Quality depends entirely on graph construction.

**Examples:** *"What are the top themes in this story?"* · *"What moral lessons does the text communicate?"*

```bash
graphrag query --root ./ragtest --method global --query "What are the top themes?"
```

Prompts: `global_search_map_system_prompt.txt` → `global_search_reduce_system_prompt.txt` → `global_search_knowledge_system_prompt.txt`

---

### Drift search

**Use for:** exploratory questions where you don't know what entity to start from, or want to discover unexpected connections.

DRIFT = **Dynamic Reasoning and Inference with Flexible Traversal**.

1. Start like a local search — find the most relevant entities for the query
2. Generate follow-up sub-questions: *"Given these entities, what else should I explore?"*
3. Search the graph for those sub-questions
4. Repeat — each iteration "drifts" deeper, following connections the LLM finds relevant
5. Synthesize all gathered evidence into a final answer

The traversal path is not fixed — it emerges from the model's reasoning at each step. Uses both `{context_data}` (local-style entity tables) and `{community_reports}` (global-style summaries) in its prompt, combining local precision with global awareness.

**Examples:** *"What unexpected connections exist between Scrooge and Christmas symbolism?"* · *"How does wealth and poverty connect across different parts of the story?"*

```bash
graphrag query --root ./ragtest --method drift --query "What connects Scrooge to Christmas symbolism?"
```

Prompt: `drift_search_system_prompt.txt`

---

### Decision guide

| Question type | Method | Why |
|---|---|---|
| About a specific person/place/event | `local` | Graph traversal from entity anchor |
| Thematic, summary, "across the whole text" | `global` | Community summaries cover the full corpus |
| Comparing two specific entities | `local` | Both become anchors, connection traversed |
| Exploratory — "what connects X to Y?" | `drift` | Iterative traversal discovers the path |
| "What does the author argue overall?" | `global` | Requires synthesizing many parts |

| Mode | Starting point | Covers | Cost |
|---|---|---|---|
| Local | Specific entities | Neighborhood | Low |
| Global | All communities | Everything | High |
| Drift | Entities → iterates | Expands as it goes | Highest |

Run local and global on the same question to understand the difference:
```bash
graphrag query --root ./ragtest --method local  --query "What is the role of money in this story?"
graphrag query --root ./ragtest --method global --query "What is the role of money in this story?"
```
Local cites specific scenes and characters. Global abstracts to themes and structure.

---

**Cost warning**

LLM is called once per chunk (extraction) + once per entity with multiple descriptions (merge) + once per community (summary). A 100-page document with 300-token chunks ≈ 500 LLM calls for extraction alone. Prompt tuning is cheap (15 chunks by default). Drift search is expensive at query time (multiple LLM calls per query).

## Relations

- [[graphrag-quickstart]] — hands-on project that runs this pipeline on A Christmas Carol

## Tags

#graphrag
