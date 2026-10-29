---
domain: knowledge-catalog
tags: [knowledge-catalog]
sources: []
updated: 2026-10-29
---

# How Knowledge Catalog Works

## What it is

A step-by-step explanation of [[knowledge-catalog]]: the problem it solves, its parts, how a question flows through them, how it captures know-how, and whether it fits a company "second brain".

In one sentence: **Knowledge Catalog is a library of cards about the company's data. People and AI fill the cards with business meaning, a human approves it, and the right few cards are handed to an AI agent at question time, so it doesn't guess.**

## What problem it solves

AI assistants can write perfect SQL and still give wrong answers, because the **meaning** of the data (what "premium customer" is, "add a $42 fee per dispute") isn't in the database. It's in documents, dashboards and people's heads. The AI fills the gap with a guess. Knowledge Catalog writes that meaning down where agents can reach it.

## Notes

### The map of this page

Each idea rests on the ones above it. Read top to bottom.

```
ROOTS    [A] data vs. "cards"        [B] an AI only knows what
             about data                  it's handed at question time
                 |                            |
                 +-------------+--------------+
                               v
LEVEL 1          [C] the problem: data has no meaning,
                     so AI guesses (premium, $42)
                               |
             +-----------------+------------------+
             v                 v                  v
LEVEL 2  [D] GATHER  ----> [E] ENRICH         [F] SERVE
         auto cards        glossary, notes,    search, context
         from many         AI proposals +      for agents,
         systems           human approval      permissions
                               |                  |
                               +--------+---------+
                                        v
LEVEL 3                  [G] Knowledge Studio + context sets
                                        |
                                        v
GOAL              [H] does it fit a document-based second brain?
```

Edges: A and B lead to C; C leads to D, E and F; D feeds E; E and F lead to G; A, C and G lead to H.

---

### [A] Data vs. cards about data

> The catalog never copies your data. It only keeps **cards** that describe it.

| | Lives where | Example |
|---|---|---|
| **Data** | In the table, the bucket, the database | Claim 123 has an excess of €300 |
| **Card** (the docs call it an *entry*) | In the catalog | "Claims table. Column `imp_fr` = excess amount. Owner: Claims team. 98% complete" |

So if someone asks *"what was the excess on claim 123?"*, the catalog doesn't answer. It tells the agent **which table** to query and **what `imp_fr` means**. The agent then queries the real data.

### [B] An AI only knows what it's handed at question time

> Everything an AI knows about **your company** when it answers comes from what is put in front of it **at that moment**.

| Source of knowledge | What it holds |
|---|---|
| **General training** | Public knowledge: what "average" means, how to write SQL, what an insurance excess is. Nothing about your company |
| **What it's handed now** | The question, plus whatever it fetches while answering: tables, documents, cards from the catalog |

**The analogy:** a brilliant temp who starts each conversation with no memory. They only know about your company what's on the desk. If a colleague told yesterday's temp "premium = 3+ policies", today's temp has never heard it.

"Memory" features (e.g. Gemini remembering something for a user) are a sticky note in **that one person's drawer**. They don't reach other users, so the rule still holds.

**Consequence:** a correction given to an AI in a chat stays **trapped in that chat**. In Bloomberg's demo, an analyst taught the agent that the fee was now $88, and the next session fell back to the $42 written in the catalog.

### [C] The problem: data has no meaning on its own

The database holds rows and column names. The meaning lives elsewhere, scattered and unwritten. In both videos, the knowledge missing from the AI's desk falls into three kinds, plus trust:

| What's missing | Example | Where it lives today |
|---|---|---|
| **1. What things are called** | 200+ tables. There's no "revenue" table, only "billing invoices" and "payment disputes" | Nowhere clear. Keyword search fails |
| **2. What business words mean** | "Premium customer", "net revenue", "severe claim" | A policy PDF, a dashboard, someone's head |
| **3. How the data must be used** | "Add $42 per dispute", "use a left join here" | A Confluence page from 2017 whose author has left |
| **Trust** | Which copy of the table is the right one? (In the first video, a Forrester analyst estimates around 30% of company data has copies) | Nowhere |

Put [B] and [C] together and a guess is guaranteed. Not because the AI is bad, but because nobody handed it the facts. As Bloomberg put it: *"the hard part isn't the SQL, it's the missing context."*

Three jobs follow:

1. **Know what exists**, so the AI finds the right table → GATHER [D]
2. **Write the meaning down** → ENRICH [E]
3. **Hand the right piece over at the right moment** → SERVE [F]

---

### [D] GATHER: cards that appear by themselves

Like a library where a card appears on its own when a book is placed on the shelf.

| Where the data lives | How the card appears |
|---|---|
| Google Cloud: BigQuery, Looker, Spanner, Bigtable, Vertex AI… | **Automatically**, as soon as it's created |
| Files in Cloud Storage (PDFs, images) | After a **discovery scan** is set up for that bucket |
| Outside Google: Oracle, MySQL, PostgreSQL, SQL Server, dbt, SAP | **Connectors** that copy only the description, never the rows |
| Other catalogs (Collibra, Atlan, DataHub…) | Partner integrations that share cards in both directions |

**On a fresh card:** only technical facts (name, columns, type, location, dates), plus:
- **Lineage:** *"table B is built from table A"*, so if A breaks you know B is affected. Kept for 30 days.
- **Usage:** who queries what, and how.

**Not on a fresh card:** meaning. The claims table's card says `severity: integer`, not what counts as "severe".

**For PDFs:** the card records that the file exists (name, size, type, folder), not what it says. The docs call it "a metadata index over unstructured data". See [[pdf-discovery-walkthrough]].

### [E] ENRICH: writing the meaning onto the cards

> Whatever enrichment reads, what it produces is always **more text on cards**: a description, a definition, a rule, a quality score.

**Three ways the text gets written:**

| Who writes | How | Example |
|---|---|---|
| 👤 **People** | **Business glossary**: the company dictionary, linked to tables and columns | "Premium customer = 3+ active policies" → customers table |
| 👤 **People** | **Notes** (the docs call them *aspects*): labelled notes attached to a card | Bloomberg's "table usage notes": *"add a $42 fee per dispute"* |
| 🤖 **AI proposes, a person approves** | Gemini writes table descriptions, guesses joins from past queries, suggests example questions with their SQL | "These two tables are usually joined on customer_id" |
| 🤖 **AI proposes, a person approves** | For Cloud Storage files (preview): summaries per folder of files, plus names and relationships pulled from the content | "This folder holds supplier invoices" |
| 🤖 **AI proposes, a person approves** | Knowledge Studio enrichment (closed preview): reads documents and past AI conversations, proposes better context | See [G] |
| 📊 **Checks** | Data quality rules, statistics, alerts for unusual values | "Alert if more than 1% of policy numbers are empty" |

**The human gate:** a data steward (the person responsible for that data being correct) sees *before → after* and accepts, edits or rejects:

```
 documents / chats / query history
            |
            v
   AI proposes: "premium = 5+ policies (was 3+)"
            |
            v
   steward reviews  --reject-->  discarded
            |
         approve
            v
   written to the catalog --> agents that read it use 5+
```

**Key nuance:** when the AI reads a Confluence page or a PDF, it reads it **to explain the data**. The document doesn't become something you can ask questions about. Its knowledge is boiled down to definitions on cards.

### [F] SERVE: the right cards, at the right moment

The AI's desk is small: it can only read a limited amount of text per question, and more text means a slower, more expensive answer.

> An agent never receives the whole catalog. It receives **the few cards that match its question**.

How a question flows:

```
"Q4 average order value, premium customers?"
        |
        v
 1. SEARCH the catalog in plain language
    -> top cards: orders table, customers table,
       glossary term "premium customer"
        |
        v
 2. FETCH the full context of those cards
    -> columns, definitions, notes, quality, joins
        |
        v
 3. AI writes the SQL and runs it on BigQuery
    (with the user's own permissions)
        |
        v
 "$88" in one turn, instead of several turns guessing
```

| Piece | Simple meaning |
|---|---|
| **Search in plain language** | "tables about claims" finds `proc_sin_sess`. Built on Google's search technology to stay fast with thousands of agents asking at once |
| **Context API / MCP tools** (preview) | The "plug" agents use to fetch a card's full context in one call. Gemini Enterprise, BigQuery's agents and your own agents connect this way |
| **Change notifications** | Systems subscribe and get told "this card changed" |
| **Data products** | A package: *"Car claims pack" = 3 tables + owner + "updated daily"* |
| **Domains** (preview) | Group data products by business area, which narrows search to the right shelf |

**Permissions:** search shows only cards for things you can access in the source system. A card never gives access to the data. Opening the table still needs normal permissions.

---

### [G] Knowledge Studio and context sets: the control room

Knowledge Studio (closed preview) is where data stewards and "context engineers" (people whose job is to curate what the AI knows) run [E] and [F] for AI agents.

| Piece | Simple meaning |
|---|---|
| **Context graph** | A visual map of tables ↔ glossary terms ↔ documents ↔ dashboards ↔ data products, with search and chat |
| **Enrichment agent** | Set up per area. Reads tables, documents (Cloud Storage, Drive, SharePoint, Confluence, GitHub) and past conversations with Gemini Enterprise and BigQuery agents. Proposes updates with a confidence score |
| **Review screen** | *Before → after* for each proposal: accept, edit, reject |
| **Context sets** | An approved, permission-controlled package for one area that agents connect to |

#### Where knowledge lives: two places

```
THE CATALOG
|
+-- TABLE CARDS (one per table, built in [D])
|     customers   [description, glossary links, notes]
|     orders
|     claims
|
+-- CONTEXT SETS (separate containers, Knowledge Studio)
      SALES set
        concept "premium = 5+ policies" ----> points to customers
        query template "AOV by segment" ----> points to orders
        document "segmentation policy"
      CLAIMS set
        concept "severe claim = ..."    ----> points to claims
```

| | Place 1: on the table's own card | Place 2: in a context set |
|---|---|---|
| **What it is** | Description, glossary links and notes attached to the table | A separate container with **its own cards**: concepts, documents, query templates, semantic models |
| **Who sees it** | Anyone with permission who looks up the table | Only agents and users **connected to that set** |
| **Who writes there** | People (Bloomberg's usage notes), Gemini descriptions | Knowledge Studio enrichment: every task **targets exactly one context set** |

The cards in a context set **point to** the tables. They don't copy them (the docs call this "zero-copy packaging"). And *"context sets do not grant data access"*: the agent still needs normal permissions on the tables.

**Example:** a rule approved into the **Sales** set reaches the Sales agent. An agent connected only to the **Service** set never sees it.

**Why it's built this way:**
- **Security:** not every agent should see every area's knowledge.
- **The small desk:** a focused package means fewer, better cards.
- **The same word means different things to different teams:** "active customer" in marketing vs. claims. Each set keeps its own approved meaning.

---

### Capturing know-how: from a chat to the catalog

Because of [B], corrections stay trapped in chats. Capturing know-how means turning them into written cards that reach every future desk.

**Three routes:**

1. **Someone writes it down.** A glossary term, a note, a quality rule. Simple, but it relies on people remembering.
2. **It's taken from conversations.** Bloomberg built their own agent for this. Knowledge Studio's enrichment agent does the same idea out of the box:

```
User corrects an agent in a chat: "the dispute fee is now $88"
        |
        v
A capture agent reads that conversation:
  the question + the SQL the agent wrote + the user's correction
        |
        v
It compares that with what the catalog says today ($42)
and works out: "the agent did X, it should have done Y"
        |
        v
It proposes a precise update:
  "payment disputes table: fee $42 -> $88"
        |
        v
A person approves it  -->  written to the catalog
                      -->  every NEW chat uses $88
```

   - The capture agent uses **the same tools** as the agent that made the mistake, so it sees the same cards and understands *why* it failed.
   - A **person approves** before anything changes. Confidence thresholds can be set, and the docs mention an option to auto-publish.
   - Bloomberg's result (their own figures): 102 usage notes across 20 tables, and 63% better SQL accuracy.

3. **It's taken from usage.** Real queries reveal which tables are usually joined, and proven question-and-SQL pairs become ready-made examples.

**Yes for data rules, no for general know-how.** Everything the capture loop produces is a card **about data**: a definition, a metric, a join rule, a warning about a table, a proven query.

| Correction in a chat | Captured? |
|---|---|
| "Exclude test policies (numbers starting with TST) from every count" | ✅ Changes how data is calculated |
| "When a customer calls about a windshield, first ask for photos" | ❌ Process know-how, no table to attach it to |
| "Be more polite in your answers" | ❌ Not about data |

---

### [H] Does it fit a document-based second brain?

| Need | Fit | Why |
|---|---|---|
| **Answers from documents** ("what does the procedure say about…?") | ❌ Not the right core tool | It's a library of cards **about data**. Documents only feed it as explanations of tables. Confluence isn't even a source it makes cards for, only something enrichment reads. It can't point to "page 14" |
| **Answers from numbers** using company definitions | ✅ Strong fit | Exactly what it's built for. Both videos are this case |
| **Capturing know-how** | ✅ for data rules, ❌ for general know-how | "Add $42" and "premium = 5+" fit. Process knowledge doesn't |

**Two example questions:**

1. *"According to our claims procedure, what documents do I request for a broken windshield claim?"* (page 14 of a Confluence procedure). At best the catalog is a signpost. Finding and reading the passage is the job of a RAG system. See [[rag/index|rag]].
2. *"How many severe claims did we have in Madrid last month?"* (data in BigQuery, definition in a policy PDF). Fits well: the definition is written in the glossary, or proposed by enrichment from the PDF and approved by a steward. The agent then gets the right tables and the official rule.

**A realistic setup: two brains that complement each other**

```
                 a question to the company's AI
                  /                          \
     "what does the procedure say?"     "how many / how much?"
                 |                              |
      DOCUMENT BRAIN (RAG)              DATA BRAIN
      searches and reads Confluence,    Knowledge Catalog: right tables
      SharePoint, PDFs                  + official definitions -> BigQuery
```

The second video shows Gemini Enterprise **connecting to** Knowledge Catalog for the data side. Gemini Enterprise and Vertex AI Search are Google's products for document search. They weren't evaluated here.

### Status and caveats (as of 2026-10-29)

| Item | Status |
|---|---|
| Rename | Dataplex Universal Catalog became Knowledge Catalog on 2026-04-10. API, CLI and roles still say `dataplex` |
| Natural-language search, data products, Cloud Storage discovery | GA |
| Context API (LookupContext), remote MCP server, domains | Preview |
| Knowledge Studio, enrichment tasks, context sets | **Closed preview**: the organization must request access. Console only |
| Security controls in Knowledge Studio | No customer-managed encryption keys, VPC Service Controls or Assured Workloads. For personal data under GDPR, Security and Data Protection must approve first |
| Cost | Knowledge Studio has no price of its own. It bills through Gemini, BigQuery and Cloud Storage |
| Claimed results | "~4x accuracy, 1/3 less latency, up to 75% fewer tokens" come from Google's internal benchmarks shown in a product video |

### Check yourself

> [!question]- The catalog's card for the claims table says "`imp_fr` = excess amount". Someone asks "what was the excess on claim 123?". What can the catalog do?
> It points to the claims table and explains the column, but it can't answer. The answer is in the rows, which the catalog doesn't store [A].

> [!question]- Colleagues told the AI in their own chats that "premium = 3+ policies", but it isn't written anywhere the AI is connected to. A new analyst asks about premium customers. What happens?
> The AI assumes a plausible definition and answers, probably wrong. Other people's chats never reach the new analyst's desk [B].

> [!question]- 300 claims-manual PDFs have been catalogued by a discovery scan. Using only that step, which PDF covers broken windshields?
> The catalog can only list the PDFs with their names, sizes and locations. It doesn't know what they say [D].

> [!question]- A rule is approved into the Sales context set. Which agents use it?
> Only agents connected to the Sales set. An agent connected only to the Service set never sees it [G].

> [!question]- Which of these chat corrections would the capture loop turn into a proposal: "exclude TST policies from counts", "be more polite", "ask for photos first on windshield claims"?
> Only "exclude TST policies". It's the only one that changes how data is read or calculated.

### Sources used

- Docs: introduction https://docs.cloud.google.com/knowledge-catalog/docs/introduction, plus its menu pages (metadata overview, about context, Knowledge Studio, context sets, enrichment tasks, search, data products, domains, connectors, Cloud Storage discovery, lineage, security, release notes, pricing)
- Video 1: Google Cloud session on Knowledge Catalog with Bloomberg Media (the "$42 problem" and their knowledge-gap agent). Transcript shared by the user, no URL
- Video 2: Google demo of Knowledge Studio (the "premium customer" / $88 average order value example). Transcript shared by the user, no URL

## Relations

- [[knowledge-catalog]] — the reference page: what the tool does and doesn't, in table form
- [[knowledge-studio]] — more detail on enrichment tasks, context sets and preview limits
- [[pdf-discovery-walkthrough]] — hands-on proof that a discovery scan registers a PDF but doesn't read it
- [[rag/index|rag]] — the "document brain": answering questions from document content is a retrieval problem
- [[agent-knowledge/index|agent-knowledge]] — same pattern of AI proposing knowledge from interactions and humans governing it

## Tags

#knowledge-catalog
