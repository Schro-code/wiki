---
domain: agentic-system-design
tags: [agentic-system-design, rag]
sources: [ai-agent-book/chapter3.md]
updated: 2026-09-17
---

# Knowledge Updates — Keeping a Knowledge Base Current

## What it is

The operational discipline for maintaining a living knowledge base: how to add new evidence without creating chaos, how to periodically restore global coherence, and how to retire stale content safely.

## What problem it solves

A knowledge base that only appends degrades over time: the same fact scatters across files, old and new claims coexist, summaries drift from evidence, and the structure no longer fits the scale. A purely periodic rewrite is too slow — new evidence cannot take effect until the next cycle. A complete update mechanism needs both paths running in parallel.

## Notes

### The two-path architecture

```
New evidence arrives
        ↓
Incremental update      ← local, immediate, event-triggered
        +
Periodic reorganization ← global, scheduled or threshold-triggered
        ↓
Coherent, navigable knowledge base
```

Neither path is a substitute for the other. Incremental updates are timely but myopic. Periodic reorganization is comprehensive but too slow to be the only path.

---

### The three-layer separation

The most important structural decision in a knowledge update system:

| Layer | Contents | Mutability |
|---|---|---|
| **Raw evidence** | Conversations, execution trajectories, source documents, tool outputs | Append-only — never rewritten |
| **Knowledge layer** | Distilled, maintainable Markdown or code | Updated via reviewed PRs |
| **Serving layer** | Vector indexes, keyword indexes — rebuilt by CI after each merge | Reproducible derivative |

The serving layer is not the source of truth. If it is deleted or corrupted, it can always be rebuilt from the knowledge layer. The knowledge layer traces back to raw evidence. Raw evidence is permanent.

**Design consequence**: when a fact is wrong, fix it in the knowledge layer. The index is rebuilt from there — never patch the index directly.

---

### Git as source of truth

Treat the knowledge base like a codebase. Every knowledge change is a Pull Request:

- **Diff review** — explicit, auditable change with before/after
- **Version history** — full provenance of every edit
- **Rollback** — one-click revert to any prior state
- **Branch isolation** — no model writes directly to `main` or to the live retrieval index

This applies to all knowledge forms: Markdown files, user-memory files, rule documents, User-as-Code executables.

---

### Incremental updates — Proposer-Reviewer pattern

A new piece of evidence triggers an update cycle between two agents:

```
Raw evidence (new conversation / document / trajectory)
        ↓
Proposer Agent
  1. Retrieve related existing knowledge
  2. Identify conflicts, outdated content, new facts
  3. Submit smallest complete diff on a working branch
  4. Maintain links, index pages, temporal metadata, evidence refs
        ↓
Reviewer Agent (independent)
  Receives: prior knowledge + diff + raw evidence
  Checks:
    - Every new assertion is supported by raw evidence
    - Qualifiers were not dropped
    - No conflicts with other files
    - Deletion/rewrite does not go too far
  Rejects with: specific evidence pointers + line numbers (not vague requests)
        ↓
Iterate until convergence
  (max iteration count or cost budget — escalate to human if not converged)
        ↓
CI on merge:
  formatting, links, metadata, permission labels, type checks, tests
        ↓
Rebuild affected chunks and retrieval indexes from merged version
```

**The Proposer's discipline**: it does not blindly append the latest conversation. It first retrieves related existing knowledge, then adds, removes, or revises the appropriate entries. The diff should be the *smallest complete change* — not a rewrite of surrounding context.

**The Reviewer's discipline**: it audits against **raw evidence**, not against the Proposer's output. If it finds new leads while checking, it continues querying — it is an agent, not a single API call.

**Convergence budget**: if the loop has not converged after the maximum iteration count or cost ceiling, it escalates to a human reviewer. It does not pass by default.

---

### Heterogeneous models for independent review

Proposer and Reviewer should use models of similar capability from **different families**:

- Claude as Proposer + GPT as Reviewer
- DeepSeek as Proposer + Kimi as Reviewer

Different training data, preferences, and reasoning habits reduce the probability that both models make the same mistake. Similar capability ensures the Reviewer does not fall behind on complex evidence.

**Permission separation enforces independence:**
- Proposer: write to working branch only
- Reviewer: read evidence + submit review results only
- Merge workflow: only actor that updates `main` and the live retrieval index

---

### Periodic reorganization — the global view

Incremental updates accumulate local changes. Periodic reorganization steps back and reconsiders the entire knowledge system. Three core tasks:

**1. Deduplicate, retire, and merge**
Scan the full knowledge base. Identify entries that:
- Duplicate other information
- Are superseded or overly fragmented
- Differ only in wording

Delete, merge, or rewrite. Rebuild links, index pages, and directory structure at the same time. Split oversized files, merge undersized ones.

*What is removed*: the knowledge-layer representation. Raw evidence beneath it is never touched.

**2. Return to raw evidence for verification**
Do not rewrite from existing summaries alone. Early omissions and misreadings propagate from one summary generation to the next. The reorganization agent must compare the knowledge section-by-section with original conversations, trajectories, and documents.

Scan in batches (by directory, time, or topic) but maintain a **coverage checklist** — batching must eventually cover everything, not become random sampling.

**3. Resolve conflicts with applicability conditions**
When statements conflict, do not pick the newest and delete the older. Trace each claim to its original source. Determine whether the claims are separately valid under:
- Different time periods
- Different subjects, regions, or tasks
- Different preconditions

If both are valid, retain both and state their applicability conditions explicitly. If evidence is insufficient, **preserve the conflict and mark it for confirmation** — do not force a resolution.

```
Wrong:  "Policy X supersedes policy Y" → delete Y
Right:  "Policy X applies to contracts signed after 2024-01
         Policy Y applies to contracts signed before 2024-01"
```

Periodic reorganization output also goes through the Proposer-Reviewer PR cycle before hitting `main`. Large diffs can be split by directory or topic, but they share one reorganization plan and a single coverage checklist.

---

### Obsolete content detection

Stale entries that remain alongside their replacements cause contradictory answers. Two approaches:

**Metadata filtering**: attach `effective_date` and `expiration_date` to each file/chunk. The retrieval layer filters expired entries before surfacing results. Expired entries never enter the LLM context.

**Explicit deprecation**: mark the entry in its summary — "This entry was deprecated on [date] and superseded by [[new-entry]]." The entry remains for provenance; retrieval can be configured to exclude it or to surface it only when the query is explicitly historical.

Both approaches share the same principle: obsolescence is declared at the knowledge layer and enforced at retrieval, not at generation time.

---

### Permission filtering — retrieval, not generation

Multi-user and multi-tenant systems must enforce access control **at the retrieval layer**:

```
Query arrives with caller identity
        ↓
Retrieval layer filters: only surface chunks the caller is authorized to see
        ↓
Authorized chunks → LLM context
```

Once sensitive content enters the LLM context, it is difficult to guarantee it will not surface in the answer. Filtering after context injection is not a reliable control.

**Tenant isolation**: separate vector indexes per tenant. A cross-tenant query must never traverse another tenant's index. Shared indexes with metadata filters are a weaker control — a misconfigured filter can expose content.

---

### Provenance as a design requirement

Every production fact must have a traceable chain:

```
Fact in knowledge layer
  ← which raw evidence (identifier)
  ← which PR (branch, diff)
  ← which reviewer approved
  ← when merged
```

Work trajectories and review feedback are archived as text. This is not logging — it is the audit trail that makes the system correctable. When a fact turns out to be wrong, provenance tells you: where it came from, who approved it, what the reviewer saw at the time.

---

### Decision guide

```
New evidence just arrived — should it update knowledge immediately?
  Trigger a Proposer-Reviewer incremental PR.
  Do not append directly to main.

The Proposer should start by:
  Retrieving related existing knowledge first
  Then proposing the smallest complete diff

The Reviewer must check against:
  Raw evidence — not just the Proposer's output

Proposer-Reviewer loop is not converging?
  Hit the iteration/cost budget → escalate to human
  Never pass by default

Knowledge base is accumulating drift (duplicate facts, structural mismatch)?
  Trigger periodic reorganization
  Return to raw evidence — do not rewrite from existing summaries only
  Maintain a coverage checklist

Two claims conflict — which to keep?
  Trace both to raw evidence
  If valid under different conditions → keep both, state applicability
  If evidence insufficient → mark conflict for confirmation, do not force resolution

Content may be outdated?
  Attach effective/expiration dates at indexing time
  Filter at retrieval — never rely on LLM to ignore stale context

Multi-tenant system?
  Separate vector indexes per tenant
  Filter permissions at retrieval layer — not at generation
```

---

## Relations

- [[chapter3-knowledge-organization]] — how knowledge is structured (RAPTOR, GraphRAG, filesystem paradigm) before updates are applied
- [[chapter3-rag-basics]] — the retrieval pipeline (chunks, dense + sparse + reranking) that operates on the serving layer rebuilt after each PR merge
- [[chapter3-user-memory-systems]] — the same Proposer-Reviewer update pattern applied to user-specific memory

## Tags

#agentic-system-design #rag
