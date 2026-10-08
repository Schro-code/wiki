---
domain: agentic-system-design
tags: [agentic-system-design]
sources:
  - raw/What Are Agentic Design Patterns? 2026 Pattern Catalog.md
updated: 2026-09-14
---

# Agentic System Design — Landscape

## What it is

The emerging discipline of designing reliable, scalable systems built around LLM-powered agents. The equivalent of "system design" for agentic AI: structural patterns, failure modes, trade-offs, and operational concerns specific to systems where the control flow is partially decided by a model.

## What problem it solves

Traditional software engineering disciplines don't cover the unique challenges of agentic systems: non-deterministic control flow, compounding errors across multi-step loops, context window management, tool reliability, and human-in-the-loop coordination. This domain maps the vocabulary, patterns, and literature needed to reason about these systems at design level.

## Notes

### The field has no single canonical name yet

Search engines return fragmented results because the discipline is pre-canonical. The terms below are the most productive entry points as of mid-2026.

### Search terms — ordered by signal-to-noise ratio

| Term | What it returns | Best for |
|---|---|---|
| `agentic design patterns` | Structural patterns for agent loops | Architecture decisions |
| `compound AI systems` | Systems combining multiple models/components | High-level framing |
| `LLM application architecture` | Broader umbrella, includes RAG and agents | Surveying the space |
| `agentic workflows` | Concrete workflow shapes (sequential, parallel, routing) | Flow design |
| `multi-agent systems` | Coordination patterns between agents | Multi-agent topology |
| `cognitive architecture LLM` | Memory, perception, action frameworks adapted from cognitive science | Agent internals |
| `LLM systems reliability` | Failure modes, retries, validation, guardrails | Operational concerns |
| `agent orchestration patterns` | Orchestrator vs worker splits, delegation | Control flow |
| `human-in-the-loop AI` | Approval gates, interruptions, feedback loops | Safety and oversight |
| `agentic system design` | The discipline itself — growing fast | Broad reading |

**Tip:** combine `site:arxiv.org` or `site:bair.berkeley.edu` with these terms to filter out marketing noise.

### Best literature — by type

#### Foundational reads (still relevant)

| Source               | Title                                          | Why it matters                                                                                                               |
| -------------------- | ---------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| Anthropic            | *Building effective agents*                    | The clearest distillation of agent patterns with explicit trade-offs. Defines five atomic workflow patterns. Starting point. |
| Lilian Weng (OpenAI) | *LLM Powered Autonomous Agents*                | Comprehensive survey: planning, memory, tool use. Most-cited overview piece, still the conceptual foundation.                |
| Berkeley BAIR        | *The shift from models to compound AI systems* | Frames the paradigm shift — why systems, not just models, are the design unit.                                               |
| Andrew Ng            | *Agentic AI Design Patterns*                   | Four core patterns: reflection, tool use, planning, multi-agent. Good first read.                                            |

#### 2026 — guides and catalogs

| Source            | Title                                                                                                                                                                         | Why it matters                                                                                                                                                  |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Augment Code      | [*Agentic Design Pattern Catalog (2026)*](https://www.augmentcode.com/guides/agentic-design-patterns)                                                                         | Consolidates Ng's 4 + Anthropic's 5 + academic sources into 12 foundational patterns + emergent patterns with maturity ratings. Best unified catalog right now. |
| SitePoint         | [*Agentic Design Patterns: The 2026 Guide*](https://www.sitepoint.com/the-definitive-guide-to-agentic-design-patterns-in-2026/)                                               | Covers 6 composable patterns end-to-end. Central thesis: mastering composable patterns beats mastering any single framework.                                    |
| Towards AI        | [*7 Design Patterns Every AI Agent Developer Should Know in 2026*](https://pub.towardsai.net/the-7-design-patterns-every-ai-agent-developer-should-know-in-2026-c77f28b51565) | Adds Memory Management and ReAct to the canonical list. Grounded in the LangChain State of AI Agent Engineering Report 2026.                                    |
| Google Cloud      | [*Choose a Design Pattern for Your Agentic AI System*](https://docs.cloud.google.com/architecture/choose-design-pattern-agentic-ai-system)                                    | Official docs. Practical trade-off tables (e.g., parallel pattern reduces latency but raises token costs).                                                      |
| MongoDB           | [*7 Practical Design Patterns for Agentic Systems*](https://www.mongodb.com/resources/basics/artificial-intelligence/agentic-systems)                                         | Good definition of the spectrum from structured workflows to fully autonomous agents.                                                                           |
| Guillaume Laforge | [*Agentic Design Patterns* (Devoxx France 2026)](https://glaforge.dev/talks/2026/05/18/agentic-design-patterns/)                                                              | Talk, not article. Covers Hierarchical Agent Decomposition in depth — smaller models for sub-tasks, larger for coordination.                                    |

#### 2026 — books

| Source | Title | Why it matters |
|---|---|---|
| Springer (2026) | [*Architectures for Agentic AI*](https://link.springer.com/book/10.1007/978-3-032-24781-0) | Unites LLMs and RL into a single framework for autonomous systems. Fills the gap between theory and real-world application. |
| Springer (2026) | [*Agentic Design Patterns: A Hands-On Guide*](https://link.springer.com/content/pdf/10.1007/978-3-032-01402-3.pdf) | 21 patterns, one chapter each. Code examples in LangChain/LangGraph, CrewAI, and Google ADK. |
| Chip Huyen (O'Reilly, 2025) | *AI Engineering* | Full stack: eval frameworks, prompt design, agent architectures, deployment trade-offs. Agents section is dense and practical. |
| KDnuggets | [*5 Best Books for Building Agentic AI Systems in 2026*](https://www.kdnuggets.com/5-best-books-for-building-agentic-ai-systems-in-2026) | Curated list — use it to discover additional titles in the space. |

#### 2026 — research papers

| Source | Title | Why it matters |
|---|---|---|
| arXiv (Jan 2026) | [*Agentic AI: Architectures, Taxonomies, and Evaluation of LLM Agents*](https://arxiv.org/html/2601.12560v1) | Surveys interaction patterns (chain, star, mesh, explicit graphs) and compares frameworks: CAMEL, AutoGen, MetaGPT, LangGraph, Swarm, MAKER. |
| VoltAgent / GitHub | [*Awesome AI Agent Papers 2026*](https://github.com/VoltAgent/awesome-ai-agent-papers) | Weekly-updated curated list from arXiv. Covers multi-agent coordination, memory, evaluation, security. Use as a reading queue. |
| Sebastian Raschka | [*LLM Research Papers: The 2026 List*](https://magazine.sebastianraschka.com/p/llm-research-papers-2026-part1) | Annual digest. 2026 heavy on reasoning models, RL, long-context efficiency, and agent harnesses. |
| GitHub | [*ICML 2026 Agent Papers*](https://github.com/jiaxianyan/icml-2026-agent-papers) | Topic-oriented: tool-use RL, multi-agent safety, computer-use agents, multi-turn attacks. |

#### Specific concerns

| Source | Title | Why it matters |
|---|---|---|
| Hamel Husain | *Your AI product needs evals* | Evaluation is the hardest unsolved part of agentic systems. Canonical practical guide. |
| OpenTelemetry docs | Agent observability | OTel became the default wire format in 2026. Vendor-neutral tracing is now table stakes. |
| Anthropic | *Tool use and computer use docs* | Tool reliability, schema design, error recovery — often underestimated. |

### 2026 key shifts (what changed from 2025)

- **Protocol convergence** — OpenAI Agents SDK, Google's Agent2Agent (A2A) protocol, and Anthropic's MCP are moving toward interoperability. Framework lock-in is decreasing.
- **Observability became standard** — OpenTelemetry is now the default wire format for agent tracing. Tools: LangSmith, Braintrust, Arize, Honeycomb. No longer an afterthought.
- **Memory layers matured** — Mem0, Letta, and Zep are now standalone products. Agent memory is a solved infrastructure problem, not a research problem.
- **Tool-typing went mainstream** — Pydantic + JSON Schema validation of tool calls cut malformed-call rates substantially. Now considered baseline.
- **Trainable agents replacing prompt-only agents** — ICML 2026 papers show a shift to RL, trajectory-SFT, and self-evolution. Agents that improve from execution traces without parameter updates.
- **Production is the new baseline** — LangChain State of AI Agent Engineering Report 2026: 57% of AI practitioners have agents in production, 30% actively building. Design quality now directly affects business outcomes.
- **Six-layer mental model** — 2026 standard: model core → memory → tools → planner → runtime → observability/evaluation. Each layer is now independently addressable.

### Conceptual mapping to system design

For context from [[system-design/index|system-design]]:

| System design concern | Agentic equivalent |
|---|---|
| Consistency | Shared state across agents — who owns it, how it's synchronized |
| Fault tolerance | Retries, fallbacks, validation after each agent step |
| Scalability | Parallel agent subgraphs, fan-out patterns |
| Observability | Run tracing, span-level logging (LangSmith, Braintrust, Arize) |
| Latency | Minimizing LLM calls, caching, async tool execution |
| Data integrity | Structured output validation, schema enforcement |

### Key patterns (stubs for future pages)

- **Reflection** — agent critiques its own output before returning it
- **Planning** — agent decomposes a task before executing it
- **Tool use** — agent delegates sub-tasks to deterministic functions
- **Multi-agent** — orchestrator delegates to specialized worker agents
- **Human-in-the-loop** — interruption points for approval or feedback
- **Routing** — LLM decides which agent or path handles a request

## Relations

- [[system-design/index]] — the classical discipline this domain parallels
- [[harness/index]] — the scaffolding layer that implements these patterns in practice
- [[rag/index]] — RAG is often a tool inside agentic systems
- [[agent-knowledge/index]] — knowledge systems that serve as agent memory

## Tags

#agentic-system-design
