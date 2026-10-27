# Wiki — Operations Log

Append-only log. One entry per operation. Never edit existing entries.

Format: `## [YYYY-MM-DD] <type> | <title>`

---

## [2026-08-24] init | Wiki initialization

Initial structure created. Active domains: synthesis.

## [2026-08-24] maintenance | Translated all wiki documents to English

## [2026-08-24] ingest | Kubernetes — kubectl basic commands

New domain created. 1 page: kubectl-commands. Source: https://kubernetes.io/docs/tutorials/

## [2026-08-25] ingest | Kubernetes — Minikube, Kustomize and kubectl updates

2 new pages: minikube (setup, docker-env, service URL, credsStore fix), kustomize (base+overlay pattern, file structure, commands). Updated kubectl-commands with apply -k and rollout restart sections.

## [2026-08-25] ingest | Harness — AI agent harness concept

New domain created. 1 page: ai-agent-harness. Covers the core loop, what a harness provides, and the spectrum from raw API to full framework. Context: conversation about Pi (pi.dev) as an example of a thin harness.

## [2026-08-25] ingest | Systems — IPC transport mechanisms

New domain created. 1 page: ipc-transports. OS-level transports (pipe, socket, shared memory, file, signal), protocols built on top (HTTP, WebSocket, gRPC, etc.), and the RPC-over-stdin/stdout pattern used by Pi.
## [2026-08-26] ingest | Harness — Pi thin harness and write ambiguity reconciliation
## [2026-08-26] ingest | Pi Coding Agent — updated from official docs (sdk, rpc, json, tui)

## [2026-08-27] ingest | Projects — index created and indexes updated to When to look

## [2026-08-27] ingest | GraphRAG — new domain from Emil Eifrem talk

New domain created. 1 page: graph-construction (covers the full pipeline: chunking → LLM extraction → merge → Leiden → community summaries → Parquet output, plus local and global search modes and decision guide). Sources: raw/GraphRAG The Marriage of Knowledge Graphs and RAG Emil Eifrem.md, raw/GraphRAG Getting Started.md

## [2026-08-27] ingest | Projects — graphrag-quickstart project created

New project: projects/graphrag-quickstart. uv + graphrag 3.1.2, Python 3.12, standard OpenAI config. Input: first 994 lines (1/4) of A Christmas Carol from Project Gutenberg. Ready to index once GRAPHRAG_API_KEY is set in .env.

## [2026-08-31] ingest | GraphRAG — Manual Prompt Tuning

New page: graphrag/manual-prompt-tuning. Covers: the token system, full token reference for all 5 indexing and query prompts, the auto→manual workflow, common manual fixes, and settings.yaml config. Source: raw/GraphRAG Manual Tuning.md

## [2026-08-31] maintenance | GraphRAG — merged prompt-tuning, manual-prompt-tuning, drift-search into microsoft-graphrag

Deleted 3 separate pages. microsoft-graphrag.md is now the single comprehensive GraphRAG reference covering the full pipeline, prompt customization (auto and manual), and all three query modes (local, global, drift). Updated graphrag/index.md (1 page) and root index.md description.

## [2026-08-31] ingest | GraphRAG — Drift Search

New page: graphrag/drift-search. Covers: what DRIFT is, the iterative traversal mechanism, comparison table across local/global/drift, why the prompt uses {community_reports}, when to use vs avoid. No raw source — written from session explanation.

## [2026-08-30] ingest | GraphRAG — Prompt Tuning (auto_prompt_tuning)

New page: graphrag/prompt-tuning. Covers: what tuning is, where it fits in the workflow (before indexing), which 3 prompts it rewrites, document selection methods, --discover-entity-types flag, CLI usage, and settings.yaml config update. Also fixed root index.md page count for graphrag domain (was 3, correct is 2). Source: raw/GraphRAG Auto Tuning.md

## [2026-08-31] ingest | Neo4j — new domain, fundamentals page

New domain created. 1 page: neo4j-fundamentals. Covers: the three primitives (nodes, relationships, properties), labels, Cypher basics (CREATE, MATCH, RETURN, variable-depth traversal), SQL vs Neo4j traversal cost comparison, free tiers, and learning resources. Written from session explanation — no raw source.

## [2026-08-31] ingest | Neo4j — Pattern matching syntax added to neo4j-fundamentals

Updated neo4j-fundamentals with a pattern matching section: node/relationship/variable syntax table, MATCH+WHERE+RETURN example, multi-hop co-actor pattern. Source: raw/Neo4j Fundamentals Pattern Matching  Querying Graphs.md

## [2026-08-31] ingest | Neo4j — Index-Free Adjacency and the O(n) problem

New page: neo4j/index-free-adjacency. Covers: Big O notation basics, the O(n) problem in relational databases (B-tree index growth under JOIN), concrete Users+Orders example with SQL and index anatomy, scaling comparison table, how Neo4j solves it with direct memory pointers (index-free adjacency), Cypher equivalent query, and when this matters most (deep hierarchies, path finding, variable-depth traversal). Source: raw/Thinking in Graphs  Graph Thinking.md

## [2026-09-01] maintenance | Renamed systems domain to system-design

Folder renamed from systems/ to system-design/. Updated root index.md (label, description, page count), system-design/index.md (frontmatter domain, title, description), and tags.md (new #system-design tag).

## [2026-09-01] ingest | System Design — IP Addresses

New page: system-design/ip-addresses. Covers: what IP is, IPv4 vs IPv6, types (public/private/static/dynamic), special reserved ranges, DORA handshake, NAT, ifconfig interface reading, MAC vs IP, and developer relevance. No raw source — written from session explanation.

## [2026-09-01] ingest | System Design — ACID and BASE

New page: system-design/acid-and-base. Covers: ACID properties (atomicity, consistency, isolation, durability) with bank transfer and flash sale examples, isolation levels, BASE properties (basic availability, soft-state, eventual consistency), ACID vs BASE comparison table, and how ACID databases scale (vertical scaling, read replicas, sharding, connection pooling). No raw source — written from session explanation.

## [2026-09-01] ingest | System Design — DNS

New page: system-design/dns. Covers: the four server types (resolver, root, TLD, authoritative), the resolution chain, query types (recursive/iterative/non-recursive), record types (A, AAAA, CNAME, MX, NS, TXT, PTR, SRV), TTL, and developer relevance (CNAME deployments, service discovery, dig). No raw source — written from session explanation.

## [2026-09-01] ingest | System Design — CAP and PACELC

New page: system-design/cap-and-pacelc. Covers: CAP theorem (C/A/P properties, CP vs AP positions), PACELC extension (latency vs consistency in normal operation), the four PACELC positions with examples, CAP consistency vs ACID consistency distinction, developer implications, and AP design patterns (last-write-wins, quorum reads, idempotent writes). No raw source — written from session explanation.

## [2026-09-03] ingest | System Design — gRPC

New page: system-design/grpc. Covers: RPC concept (remote function call, network invisible to caller), how gRPC works (proto file → code generation → server implementation → client stub), the most common use case (microservices internal communication, REST outside / gRPC inside pattern), REST vs gRPC comparison table (format, browser support, streaming, contract enforcement, coupling), when to use each (browser → REST, controlled both sides → gRPC), workflow difference (proto file replaces Swagger docs and manual serialization), debugging with grpcurl, and why ChatGPT streaming uses SSE not gRPC. No raw source — written from session explanation.

## [2026-09-03] ingest | System Design — REST API

New page: system-design/rest-api. Covers: what REST means (Representational State Transfer broken down), the 6 constraints (uniform interface, client-server, stateless, cacheable, layered system, code on demand), stateless clarified (server stores no client session state — not that there's no state), how statelessness enables horizontal scaling (any instance can serve any request), chatbot/LangGraph example (MemorySaver vs PostgresSaver), why statelessness enables caching (deterministic responses per URL), server-side Redis caching as not a violation (per-request not per-client), JWT as the stateless auth pattern, HTTP verbs with idempotency column, HTTP status code classes, typical resource URL pattern, and trade-offs. No raw source — written from session explanation.

## [2026-09-02] ingest | System Design — Sagas

New page: system-design/sagas. Covers: what Sagas are and why the name (1987 Garcia-Molina paper, Norse saga analogy), the design (forward steps + compensating actions in pairs), happy path vs failure path, the two conditions (steps must be undoable, compensations must eventually succeed), the coordinator problem and why durable state persistence solves it, concrete Azure implementation with Azure Durable Functions (replay-based execution, idempotency requirement, event log anatomy), orchestration vs choreography, and when to use Sagas vs 2PC vs batch cleanup. No raw source — written from session explanation.

## [2026-09-02] ingest | System Design — Distributed Transactions

New page: system-design/distributed-transactions. Covers: what a transaction is, the 6 transaction states in a single database (active, partially committed, committed, failed, aborted, terminated), why distributed transactions are harder, Two-Phase Commit (prepare + commit phases, coordinator role, blocking problem), Three-Phase Commit (pre-commit phase, how it breaks the blocking deadlock), Sagas as the practical microservices alternative, and the developer rule of thumb (avoid the problem first). No raw source — written from session explanation.

## [2026-09-03] lint | Inconsistency sweep

5 fixes applied:
- system-design/ipc-transports.md: domain and tags updated from `systems` → `system-design` (missed during 2026-09-01 domain rename)
- root index.md: neo4j page count corrected 1 → 2
- raw/index.md: added 2 unregistered source files (Neo4j Fundamentals Thinking in Graphs, GraphRAG Prompt Tuning Overview)
- neo4j/index-free-adjacency.md: normalized tags to inline form `[neo4j]` to match convention
- tags.md: removed orphaned `#systems` tag (no page uses it after the ipc-transports fix)

## [2026-09-07] ingest | RAG — SPLADE page created

New page: rag/splade. Covers: query expansion with concrete insurance example (roban→sustracción), BERT dual-encoder architecture, training on (query, doc) pairs, sparsity regularization loss (retrieval_loss + λ × sparsity_penalty), why sparsity penalty is needed (BERT outputs dense by default), dynamic corpus advantage (no IDF recomputation), SPLADE vs TF-IDF vs BM25 vs dense comparison tables, when to use and avoid, benchmark context (TF-IDF used, SPLADE is the natural upgrade). Updated rag/index.md.

## [2026-09-07] ingest | RAG — TF-IDF page created

New page: rag/tf-idf. Covers: TF × IDF formula, IDF table with 382-chunk examples, sparse vector representation (vocabulary dimensions, dot product similarity), why it avoids the stemming problem, TF-IDF vs BM25 comparison (what BM25 fixes), dynamic corpus problem with three strategies (periodic recomputation, incremental approximation, frozen vocabulary), when to use. Updated rag/index.md.

## [2026-09-07] ingest | RAG — BM25 page created

New page: rag/bm25. Covers: building BM25 from intuition (raw count → saturation → length normalization), TF_bm25 formula with saturation table (k1=1.2, ceiling at k1+1), IDF formula with concrete example across 382 chunks, full BM25 formula for multi-word queries, Spanish morphology breakdown (stemming failure, `cubrir` surface forms), BM25 vs TF-IDF vs SPLADE comparison table, when to use. Updated rag/index.md.

## [2026-09-07] ingest | RAG — hybrid-search hub page created

New page: rag/hybrid-search. Covers: the full hybrid search landscape (dense single-vector, dense multi-vector/ColBERT, sparse TF-IDF/BM25/SPLADE, fusion RRF/linear/learned), dense vs sparse comparison table, RRF formula with worked 3-chunk example, BM25 and SPLADE overviews (deep-dive pages to create), benchmark results table, strategy selection guide. Placeholder wikilinks for tf-idf, bm25, splade pages to be created. Updated rag/index.md.

## [2026-09-07] ingest | RAG — ColBERT page created

New page: rag/colbert. Covers: the negation problem with pooled embeddings, one-vector-per-token architecture, MaxSim operator with worked numeric example (2.74 vs 2.00 on "no cubre" vs "sí cubre"), computational cost (6,064 comparisons per document vs 382 for dense), two-stage retriever vs re-ranker design, recall ceiling problem, words-vs-tokens context limit rule, benchmark results (negation +29.7%, comparison +35.6%), when to use (measure first, don't add speculatively). Updated rag/index.md.

## [2026-09-07] maintenance | RAG — hybrid search, filtered search, and ColBERT sections expanded

Added to rag-fundamentals: RRF formula with concrete 3-chunk worked example, controlled vocabulary insight (why hybrid gains are large for legal/insurance text), BM25 vs TF-IDF sparse explanation (language-agnostic corpus-trained vs English-tuned general algorithm), Vertex AI filter failure mechanism (LLM-generated filters vs exact token restricts), "worse than no filter" rule with concrete numbers (0.131 < 0.817), ColBERT decision framework (measure negation recall first, don't add speculatively).

## [2026-09-07] maintenance | RAG — rag-fundamentals rewritten as knowledge document

Rewrote rag-fundamentals.md from project report to self-contained knowledge document. Added: full chunking strategy comparison (character/sentence/paragraph/sliding window/semantic/LLM — with trade-offs and when to discard each), metadata design methodology (start from user queries), worked numeric examples for all 4 metrics (Recall@k, Precision@k, MRR, nDCG with DCG/IDCG calculation), ColBERT explanation from first principles (pooled vs token-level embeddings, MaxSim). Removed project-specific framing, kept benchmark data as concrete examples.

## [2026-09-07] ingest | RAG — new domain, rag-fundamentals page

New domain created. 1 page: rag-fundamentals. Covers the full RAG pipeline built for the lda-vectordb-benchmark: chunking (512 words, 64 overlap, <30 discard), metadata schema (article_references, monetary_values, coverage_types), multilingual-e5-large embeddings and E5 prefix system, LLM-generated ground truth (2,690 queries, 5 types), retrieval metrics (Recall@k, MRR, nDCG), pure vector / hybrid / filtered search comparison across 4 databases, ColBERT late interaction results, and use-case recommendations. Updated root index.md and tags.md (#rag). Source: projects/lda-vectordb-benchmark README.

## [2026-09-07] ingest | Projects — lda-vectordb-benchmark project registered

## [2026-09-08] ingest | agent-knowledge — new domain, 5 pages: bitemporal-knowledge-graph, ontology, knowledge-pipeline, knowledge-governance, agent-query-loop
## [2026-09-08] maintenance | agent-knowledge — consolidated 5 pages into single utopia.md reference

New project: projects/lda-vectordb-benchmark. Comparative benchmark of 4 vector databases (Azure AI Search, Vertex AI Vector Search, Pinecone Local, Qdrant) using 11 Spanish insurance PDFs (Coche, Hogar, Moto). Dataset: 382 chunks, 2,690 synthetic queries (semantic/hybrid/filtered/negation/comparison), embeddings pre-computed with multilingual-e5-large (1024 dims). Metrics: Recall@k, Precision@k, MRR, nDCG, CRUD latency, write-to-read consistency. Key findings: Qdrant+ColBERT achieves highest retrieval quality (R@10=0.953), Azure AI Search is cheapest ($75/mo) with best filtered recall (0.929), Vertex AI has the slowest setup (54 min). Updated projects/index.md.

## [2026-09-10] maintenance | System Design — fd foundation section added to ipc-transports

Added "Foundation: how the OS hands you a channel" section to ipc-transports.md. Covers file descriptors inline (front door vs. private room, ulimit) so the page is self-contained, with a link to file-descriptors for the full deep-dive.

## [2026-09-10] maintenance | System Design — file-descriptors rewritten for clarity

Rewrote file-descriptors.md with simpler framing: why the server must ask the OS, the front-door vs. private-room mental model (listening socket stays open, accept() creates a dedicated fd per client), full step-by-step connection flow diagram, and a clearer explanation of why the fd limit equals the connection limit.

## [2026-09-09] ingest | System Design — file-descriptors page created

New page: system-design/file-descriptors. Covers: what an fd is (integer handle to a kernel-managed resource), the per-process fd table (0=stdin, 1=stdout, 2=stderr), "everything is a file" (files, sockets, pipes, devices all use the same fd API), fd limits (soft/hard per-process, system-wide), how to tune with ulimit, and why connections consume fds. Updated system-design/index.md and added cross-reference in ipc-transports.md.

## [2026-09-09] maintenance | System Design — ipc-transports expanded

Rewrote ipc-transports.md with a two-layer model (Layer 0: OS mechanisms with locality property; Layer 1: protocols that inherit that locality). Added: locality concept explained clearly with full table, updated protocols table, ASCII mental model diagram, system design implications of crossing the network (latency/partial failure/ordering), mechanism selection guide, and cross-references to cap-and-pacelc, distributed-transactions, sagas, and grpc.

## [2026-09-14] ingest | System Design — rate-limiter page created

Created rate-limiter.md covering: placement options (client/server/middleware), throttle rules, all 5 algorithms (Token Bucket, Leaking Bucket, Fixed Window Counter, Sliding Window Log, Sliding Window Counter) with comparison table, high-level architecture (Redis counters, cached rules, workers, drop vs queue options), distributed challenges and solutions, and client-facing headers. Source: liquidslr/system-design-notes chapter 4.

## [2026-09-14] ingest | System Design — consistent-hashing page created

Created consistent-hashing.md covering: the rehashing problem with normal modulo hashing, the hash ring as a wrapped number range, O(log N) lookup via binary search on a sorted array, adding/removing servers and which keys are affected, virtual nodes as the solution to uneven distribution, and real-world users (DynamoDB, Cassandra, Discord, Akamai, Maglev). Mermaid diagrams used throughout for Obsidian rendering. Source: liquidslr/system-design-notes chapter 5.

## [2026-09-14] maintenance | Agentic System Design — overview updated with 2026 sources

## [2026-09-14] ingest | Agentic System Design — domain created with overview

Created new domain `agentic-system-design/`. Added `overview.md` mapping the field: 10 search terms ordered by signal-to-noise, curated literature in three tiers (foundational, engineering depth, specific concerns), conceptual mapping to system design, and stubs for the key patterns to expand later. Updated `Wiki/index.md`, `Wiki/tags.md` with new domain and tag.

## [2026-09-15] refactor | Renamed projects/ → labs/, created projects/ for work knowledge bases

Renamed `projects/` to `labs/` to disambiguate hands-on learning labs from work project knowledge bases. Created new `projects/` domain for storing meetings, notes, decisions and context per work project. Updated all references: `Wiki/index.md`, `Wiki/tags.md`, `Wiki/CLAUDE.md`, `Wiki/labs/index.md`, `Wiki/labs/graphrag-quickstart/graphrag-quickstart.md`, `Wiki/rag/index.md`. Log entries left untouched (append-only rule).

## [2026-09-15] lint | Structural fixes from soundness audit

5 fixes applied:
- raw/index.md: registered missing source `What Are Agentic Design Patterns? 2026 Pattern Catalog` (capture date 2026-09-14); updated `updated` date from 2026-08-28 → 2026-09-15
- agentic-system-design/overview.md: set `sources: [raw/What Are Agentic Design Patterns? 2026 Pattern Catalog.md]` (was empty despite using that source)
- labs/index.md: added missing `utopia` lab (deeplethe/utopia — bitemporal knowledge graphs, ontology, governance, Rust, MCP)
- system-design/index.md: bumped `updated` from 2026-09-09 → 2026-09-15 (rate-limiter and consistent-hashing were added 2026-09-14 but date was never updated)

Remaining open items (require raw content capture):
- rate-limiter.md and consistent-hashing.md list GitHub URLs in `sources:` instead of raw/ file paths — sources need to be captured to raw/ first
- agent-knowledge/utopia.md has no raw/ source (fed from project code, not a captured document)

## [2026-09-15] ingest | System Design — http-grpc-websocket page created

## [2026-09-17] ingest | Agentic System Design — AI Agent Book (bojieli, full book-en)

Copied all 13 markdown files (introduction, chapters 1–10, afterword, reference-answers) from github.com/bojieli/ai-agent-book/book-en into agentic-system-design/ai-agent-book/. Created index.md with chapter titles. Source: https://github.com/bojieli/ai-agent-book/tree/main/book-en

## [2026-09-16] ingest | Agentic System Design — User Memory Systems

New page: agentic-system-design/user-memory-systems. Covers: two memory tiers (trajectory vs long-term), three-level evaluation framework (L1 Basic Recall, L2 Multi-Session Retrieval, L3 Proactive Service), four storage formats (Simple Notes → Enhanced Notes → JSON Cards → Advanced JSON Cards) with selection guide, three cognitive types (episodic/semantic/procedural), Mem0 v2 vs v3 (write-time vs retrieval-time conflict resolution, append-only store), Memobase (profile slots + event timeline, buffered batch), three-tier compression (importance scoring → clustering → abstraction), and PII sanitization with local model. Source: Chapter 3.1, ai-agent-book (github.com/bojieli/ai-agent-book). Updated agentic-system-design/index.md.

## [2026-09-17] ingest | Agentic System Design — RAG Basics
New page: agentic-system-design/rag-basics. Covers: RAG core flow (retrieve → augment → generate), three chunking strategies (fixed-size, recursive/structure-aware, semantic) with size trade-offs and the context-loss flaw, dense embeddings and cosine similarity, embedding model evolution (Word2Vec → BERT → BGE-M3), ANN index choice (ANNOY vs HNSW). Updated agentic-system-design/index.md. Sparse embeddings and hybrid retrieval pending (chapter not fully pasted yet).

## [2026-09-17] ingest | Agentic System Design — Knowledge Organization (Section 3.3)

New page: agentic-system-design/knowledge-organization. Covers: naive RAG failure modes (aggregation blindness, missing boundary semantics), RAPTOR bottom-up tree hierarchy (cluster → summary → recurse → root), GraphRAG entity-relationship graph (multi-hop reasoning, entity disambiguation, triple extraction limitations), OpenViking filesystem paradigm (L0/L1/L2 abstraction layers, progressive disclosure), and the mandatory linking requirement (Wikipedia-style cross-references, explicit prompt requirement). Decision guide: when to use structured indexing vs hybrid retrieval. Updated agentic-system-design/index.md. Source: Chapter 3.3, ai-agent-book.

## [2026-09-17] ingest | Agentic System Design — Knowledge Updates (Section 3.3.3)

New page: agentic-system-design/knowledge-updates. Covers: two-path update architecture (incremental + periodic), three-layer separation (raw evidence append-only / knowledge layer PR'd / serving layer rebuilt), Git as source of truth, Proposer-Reviewer multi-agent pattern (smallest complete diff, audit against raw evidence, convergence budget, human escalation), heterogeneous model review (different families to reduce correlated errors), permission separation by role, periodic reorganization (deduplicate/retire/merge, verify against raw data not summaries, conflict resolution with applicability conditions), obsolete content detection (effective/expiration date metadata, filter at retrieval), permission filtering at retrieval layer (not generation), and provenance tracking. Updated agentic-system-design/index.md. Source: Chapter 3.3.3, ai-agent-book.

## [2026-09-18] update | OpenViking lab — local project registered
Created labs/openviking.md pointing to /Users/LDAZZH/Desktop/Proyectos/OpenViking/. Updated labs/index.md. Added local path reference to agentic-system-design/knowledge-organization.md (Approach 3 section).

## [2026-09-18] update | OpenViking lab — implementation deep-dive
Read source: ClaudeCodeSource, SessionReplayer, Context/ContextLevel, MemoryConfig, extractor.py, namespace.py, ragfs Rust crate. Updated labs/openviking.md with full implementation section: two ingestion pipelines (conversations vs PDFs), ContextLevel enum confirming L0/L1/L2, URI namespace mapping, memory extraction at commit time, ragfs backends, ingest adapter table.

## [2026-09-18] meeting | lupe-v2 — sprint review eficiencia septiembre

## [2026-09-21] ingest | Agentic System Design — RLHF and RLVR

New page: agentic-system-design/rlhf-rlvr. Covers: SFT as the prerequisite stage, reward model construction (head swap, comparison pairs), PPO loop with KL penalty and ratio trick, RLVR replacing the reward model with a verifier (binary reward), emergent chain-of-thought reasoning from RLVR, GRPO vs PPO, and relation to agentic systems (alignment vs reasoning, agent RLVR). No raw source — written from session explanation. Updated agentic-system-design/index.md.

## [2026-09-21] ingest | Agentic System Design — Agentic RAG, Contextual Retrieval, Two-Tier Memory, Knowledge Discovery

## [2026-09-21] maintenance | Agentic System Design — chapter3 prefix added to all chapter 3 pages

Renamed 8 pages in agentic-system-design/ with chapter3- prefix: user-memory-systems, rag-basics, knowledge-organization, knowledge-updates, agentic-rag, contextual-retrieval, two-tier-memory-architecture, knowledge-discovery-from-data. Updated agentic-system-design/index.md, labs/openviking.md, and rlhf-rlvr.md. All internal cross-references updated.

## [2026-09-21] ingest | Agent-Native — BuilderIO agentic app framework

## [2026-09-22] meeting | 3 meetings resolved from staging

3 meetings processed: ia-voz/sprint-review-siniestros-voz (2026-09-18), ia-dailies/daily-standup (2026-09-21), lupe-v2/sprint-sync-genesis-deep-links (2026-09-21). 1 empty recording (752348, 1 second) deleted. Staging queue now empty.

New lab page: agent-native. TypeScript framework for building apps where agent and UI share the same action layer (defineAction). Source: https://github.com/BuilderIO/agent-native

## [2026-09-23] meeting | 3 meetings resolved from staging

3 meetings processed: ia-dailies/daily-standup (2026-09-22), lupe-v2/sync-q4-deeplinks-dashboard (2026-09-23), ia-dailies/daily-standup (2026-09-23). Staging queue now empty.

## [2026-09-22] ingest | Labs — MeloTTS Chinese TTS pipeline

New lab: melotts-chinese. Chinese text-to-speech pipeline using MeloTTS (myshell-ai/MeloTTS-Chinese). Python project at Wiki/labs/melotts-chinese/ — uv setup, main.py with synthesize() helper, model downloads automatically from HuggingFace on first run.

## [2026-09-22] maintenance | Labs — melotts-chinese extended to English TTS

Added English TTS support (myshell-ai/MeloTTS-English). New tts.py shared module with cached model loading. main.py now demos 3 Chinese + 3 English sentences (EN-Default, EN-US, EN-BR). One-time setup: averaged_perceptron_tagger_eng NLTK corpus.

## [2026-09-24] ingest | Labs — speech-to-speech (HuggingFace voice agent pipeline)

New lab created. Cloned huggingface/speech-to-speech into labs/speech-to-speech. Set up venv with Python 3.12 + uv. Installed package. Pipeline ran successfully (Apple Silicon fully local): downloaded Parakeet TDT STT + Qwen3-4B-Instruct-4bit LLM via MLX. TTS model (Qwen3-TTS) downloads on next full interactive run. Note file: labs/speech-to-speech.md.

## [2026-09-22] ingest | TTS — new domain, tts-pipeline page

New domain created. 1 page: tts-pipeline. Covers the full TTS pipeline from first principles: what digital audio is (24kHz samples, speaker physics), G2P as the tokenizer equivalent, FFT and frequency bins, mel spectrogram (windows × 80 matrix, filterbank, log), acoustic model, vocoder (why phase is lost and how the vocoder fills it in), training on (text, audio) pairs, and Kokoro-82M specifics (StyleTTS2, style vector, KPipeline streaming). Written from session explanation — no raw source.
## [2026-09-28] meeting | 2 meetings resolved from staging (lupe-v2, ia-voz) + 3 non-meeting captures deleted

## [2026-09-30] ingest | RDF, OWL y Turtle — nueva página en agent-knowledge

Nueva página rdf-owl-turtle.md en agent-knowledge. Cubre: diferencia RDF vs OWL, Turtle como sintaxis, cómo identificar clases vs instancias, las 4 secciones de un .ttl exportado de Utopia, puntuación Turtle, sistema de prefijos, URNs, y la señal visible de la rotura de auto-creación de relaciones. Escrito desde sesión de exploración con el export real de la KB.

## [2026-10-08] ingest | Inference Engineering — new domain created

## [2026-10-08] ingest | OpenAI API standard — new page in inference-engineering

## [2026-10-09] lint | Full ritual audit — 18 issues found, 17 fixed

Fixes applied:
- 4 broken wikilinks repaired (graph-construction→microsoft-graphrag ×2, agentic-rag→chapter3-agentic-rag, ../rag/graphrag→graphrag/microsoft-graphrag)
- 9 phantom pages removed from inference-engineering/index.md (listed but never created)
- 3 tag format fixes (removed # prefix from YAML tags in rdf-owl-turtle, openai-api-standard, roadmap)
- Registered #book tag in tags.md for ai-agent-book reference material
- Root index.md updated date corrected (2026-09-07 → 2026-10-09)
- Template headings standardized: acid-and-base, cap-and-pacelc ("What they are/solve" → "What it is/solves")
- Missing "What problem it solves" sections added to kubectl-commands and minikube
- coding-agent-inference-infra lab registered in labs/index.md
- 6 lab wiki note files created (guardian-agent, lda-vectordb-benchmark, terraform, terraform-tutorial, terraform-azure, learn-terraform-docker-container)

Open items not fixed:
- projects/utopia/ empty folder (deletion denied by user)
- consistent-hashing and rate-limiter sources: URLs instead of raw/ paths (requires fetching external content)
- 8 chapter3-*.md sources reference ai-agent-book/chapter3.md instead of raw/ paths (book is embedded reference material, not in raw/)
- 3 unreferenced raw/ files (GraphRAG Prompt Tuning Overview, Software Engineering Fundamentals, System Design)
- 14 raw/ files use YAML frontmatter instead of plain Source: header
- Extra top-level ## sections in dns, ip-addresses, microsoft-graphrag (structural preference, content is correct)
- Content spread across custom ## sections in ipc-transports, utopia, roadmap (structural preference, content is correct)

## [2026-10-09] ingest | local-inference-stack lab created in labs/

## [2026-10-14] ingest | SCM — new domain created

New domain: scm (Software Configuration Management). 1 page: scm.md. Covers: what SCM is, key subdomains (branching strategies, fork management, release engineering, patch series, supply chain), fork management patterns comparison table (merge-based, rebase-based, layered architecture, vendor branch), key principles (minimize delta, merge don't cherry-pick, upstream-first, layered customization), and key references. Written from session conversation about maintaining the LDA fork of deeplethe/utopia.

## [2026-10-14] ingest | SCM — fork-management page created

New page: scm/fork-management. Covers: vocabulary (upstream, downstream, delta, carry patches), the three sync approaches (merge-based, rebase-based, layered architecture) with diagrams, cherry-pick as anti-pattern, 7 tactics to reduce delta (upstream-first, runtime config, composition, override files, build args, extension points, git attributes), the tactic ladder summary, choosing an approach by situation.

## [2026-10-15] ingest | SCM — fork-development-workflow page created

## [2026-10-16] ingest | DevOps — new domain, 5 pages

New domain: devops. Pages created: devops-overview (culture, Three Ways, CALMS, DORA metrics, DevOps vs SRE vs Platform Eng), branching-strategies (Gitflow vs GitHub Flow vs trunk-based, user's dev/main model situated), ci-cd-pipeline (all pipeline stages beyond lint+test, pipeline-as-code, artifact principle, branch-stage mapping), environments (dev/staging/prod purposes, parity, ephemeral envs, config injection), deployment-strategies (recreate, rolling, blue-green, canary, feature flags, expand-contract DB migrations). Updated Wiki/index.md and Wiki/tags.md.

## [2026-10-19] ingest | Testing — new domain, 5 pages (test-pyramid, end-to-end-testing, smoke-testing, integration-testing, learning-plan)

## [2026-10-20] ingest | Testing — flakiness page created

New page: testing/flakiness. Covers: what flakiness is, the five root causes (timing, shared state, test data, external dependencies, animations/transitions) each with concrete examples and fixes, how causes compound, and the retries trap. Includes 5-question quiz. Corresponds to node 3 of the testing learning plan.

## [2026-10-20] ingest | Testing — domain refactor: 2 new pages, 4 diagrams, learning path consolidation

Refactored the testing domain. New pages: unit-testing (base layer, isolation, Arrange-Act-Assert, speed advantage), test-doubles (taxonomy: dummy/stub/spy/mock/fake, state vs behavior verification, over-mocking trap). Added Mermaid diagrams to: test-pyramid (block-beta pyramid), end-to-end-testing (sequence diagram of browser mechanics), smoke-testing (deployment pipeline flowchart with gates), flakiness (diagnostic decision tree for 5 root causes). Absorbed learning-plan into test-pyramid as a "Learning path" section. Removed learning-plan from index. Updated cross-references across all pages.

## [2026-10-20] ingest | DevOps — terraform page created

New page: devops/terraform. Learning guide covering Terraform fundamentals in three phases: reading HCL configs (file conventions, terraform/provider/resource/data/variable/output blocks, referencing syntax, reading plan output), the mental model (declarative vs imperative, dependency graph, CRUD lifecycle, mutable vs immutable changes, drift detection), and commands (init, plan, apply, destroy). Includes self-assessment quiz. Registered #iac and #terraform tags.

## [2026-10-21] ingest | Archify — new domain, 4 pages

New domain: archify. Pages created: two-stage-pipeline (Claude writes JSON spec → Node.js renderer → HTML, separation of concerns, diagnostics), json-schema-contract (12 schemas as contract between Claude and renderer, validation, capability ceiling), diagram-type-selection (10 types, question-based selection heuristic, workflow vs sequence distinction), explainer-mode (same two-stage pattern for full-page documents). Updated Wiki/index.md and Wiki/tags.md (#archify).

## [2026-10-26] ingest | Neuroscience — new domain, 6 pages (reward-prediction-error, temporal-difference-learning, blocking-and-credit-assignment, dopamine-wanting-vs-liking, rpe-open-questions, rpe-and-habit-change)

New domain: neuroscience. Built from a tutoring session plus a limited fact-check (web search was blocked, so Europe PMC and Wikipedia were used); no raw/ sources, so `sources` is empty and each page carries an evidence note separating verified claims from ones taken from memory. Pages created: reward-prediction-error (signed surprise, update rule, canonical dopamine recordings, motivation link), temporal-difference-learning (the error moves back to the earliest predictor), blocking-and-credit-assignment (redundant cues earn no credit, Rescorla-Wagner with summed prediction), dopamine-wanting-vs-liking (dopamine is not happiness), rpe-open-questions (movement and motivation, ramps, novelty, weak negative coding, extrapolation to daily life), rpe-and-habit-change (reasoned hypotheses with an evidence-status table). Updated Wiki/index.md and Wiki/tags.md (#neuroscience, #reward-learning). Candidate pages: reinforcement learning, distributional RL.

## [2026-10-27] ingest | Neuroscience — effort conditioning and discipline playbook; domain consolidated from 6 pages to 3

Source: a popular video on discipline (Psych Tunes) worked through in a tutoring session, with a research-subagent fact-check of the cited studies (Europe PMC was unavailable and web search was blocked, so Crossref, OpenAlex, PubMed abstracts and two PMC full texts were used); no raw/ file, so `sources` stays empty. New pages: effort-conditioning (effort is felt, learned industriousness, peak-end rule, why effort got trained as a cost, three rewiring rules: immediate, contingent, repeated) and discipline-playbook (five techniques with reasons and costs, evidence map, backfire risks such as Deci 1999, two-week experiment protocol, plus the older RPE habit hypotheses). Restructured: reward-prediction-error now absorbs temporal-difference-learning, blocking-and-credit-assignment, dopamine-wanting-vs-liking and rpe-open-questions as numbered sections; rpe-and-habit-change was merged into discipline-playbook. Removed 5 files (recoverable from commit 1bca90b). Updated Neuroscience/index.md, Wiki/index.md and Wiki/tags.md (#habits). Corrections recorded: Eisenberger 1992 is in Psychological Review; Lally's 66-day median and Hofmann's "weaker desires" are unverified. Candidate pages: overjustification effect, habit formation (Lally), temptation bundling, reinforcement learning, distributional RL.
