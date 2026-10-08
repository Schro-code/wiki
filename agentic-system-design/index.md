---
domain: agentic-system-design
type: index
updated: 2026-09-17
---

# Agentic System Design — Index

Design discipline for LLM-powered agent systems: patterns, failure modes, trade-offs, and operational concerns. Read `overview.md` first to orient yourself in the field.

## Pages

| Page | When to look | Sources |
|---|---|---|
| [[overview]] | Orienting yourself in the field: search terms, best literature, conceptual map, key patterns | — |
| [[chapter3-user-memory-systems]] | Look when designing cross-session personalization: storage formats, memory tiers, compression, privacy, framework comparison (Mem0 vs Memobase) | [[ai-agent-book/chapter3\|chapter3]] |
| [[chapter3-rag-basics]] | Look when building a knowledge acquisition pipeline: chunking strategies, dense embeddings, ANN indexes (ANNOY vs HNSW), and the path to hybrid retrieval | [[ai-agent-book/chapter3\|chapter3]] |
| [[chapter3-knowledge-organization]] | Look when naive RAG is failing (aggregation blindness, missing boundary rules) or when queries require cross-document synthesis: RAPTOR tree index, GraphRAG entity-relationship graph, filesystem paradigm, mandatory linking | [[ai-agent-book/chapter3\|chapter3]] |
| [[chapter3-knowledge-updates]] | Look when designing the update pipeline for a knowledge base: Proposer-Reviewer multi-agent PR pattern, Git as source of truth, three-layer separation (raw evidence / knowledge / serving), periodic reorganization, conflict resolution, obsolete content, permission filtering at retrieval | [[ai-agent-book/chapter3\|chapter3]] |
| [[ai-agent-book/index\|ai-agent-book]] | Reading the bojieli AI Agent Book (10 chapters: context engineering, tools, coding agents, evaluation, multi-agent, etc.) | [github.com/bojieli/ai-agent-book](https://github.com/bojieli/ai-agent-book) |
| [[rlhf-rlvr]] | Look when you want to understand how LLMs are trained after pretraining: RLHF pipeline (SFT → reward model → PPO), RLVR with verifiable rewards, GRPO, and how these techniques produce reasoning and aligned agents | — |
| [[chapter3-agentic-rag]] | Look when designing retrieval that must handle multi-hop, complex questions: ReAct loop for retrieval, RAG-as-tool, agentic vs non-agentic comparison, security boundaries (prompt injection, knowledge poisoning) | [[ai-agent-book/chapter3\|chapter3]] |
| [[chapter3-contextual-retrieval]] | Look when retrieval quality is suffering from context-loss in chunked documents or contradictory memory records: LLM-generated prefix technique, 49%/67% failure rate reduction, how it differs from contextual compression | [[ai-agent-book/chapter3\|chapter3]] |
| [[chapter3-two-tier-memory-architecture]] | Look when designing a memory system that needs proactive service (L3): JSON Cards resident in context (overview) + contextual RAG on demand (details), and how they converge to achieve what neither achieves alone | [[ai-agent-book/chapter3\|chapter3]] |
| [[chapter3-knowledge-discovery-from-data]] | Look when the knowledge you need is implicit in structured datasets, not in documents: two-phase extraction pipeline (LLM structuring → factor importance clustering), bottom-up schema discovery, one-hot encoding, case prototypes | [[ai-agent-book/chapter3\|chapter3]] |

