---
domain: archify
tags: [archify]
sources: []
updated: 2026-10-21
---

# Diagram Type Selection

## What it is

A decision framework for choosing the right archify diagram type. Archify has 10 types, each with its own schema and renderer, because each solves a fundamentally different layout problem.

## What problem it solves

Users don't know which diagram type to ask for. The 10 types feel arbitrary until you realize each one answers a specific question — match your question to the type and it picks itself.

## Notes

### The rule

**Each diagram type answers one specific question.** Phrase what you're trying to explain as a question, and the type reveals itself.

| Your question | Type |
|---|---|
| What are the parts and how are they connected? | **architecture** |
| What are the steps to complete this process? | **workflow** |
| Who talks to whom, in what order? | **sequence** |
| How does data transform as it flows through? | **dataflow** |
| What states can this be in, and what triggers changes? | **lifecycle** |
| What entities exist and how do they relate? | **erd** |
| What types/classes exist and what do they contain? | **class** |
| What's the hierarchy? | **tree** |
| What happened when? | **timeline** |
| How long did each step take? | **waterfall** |

### Workflow vs sequence

The most common confusion. Both show things happening in order, but the units differ:

- **Workflow** — units are **steps in a process**. Decisions, branches, loops. "Fill form → validate → decide → submit." Can be drawn as a flowchart with decision diamonds.
- **Sequence** — units are **messages between participants**. Distinct actors/services exchanging calls and responses. "Gateway calls Auth. Auth responds. Gateway calls Orders." Can be drawn with vertical lifelines and horizontal arrows.

**Heuristic: "Am I explaining a process, or a conversation?"** Process → workflow. Conversation → sequence.

### Why 10 types, not one flexible type

Each type needs its own layout algorithm. The math for placing boxes and routing arrows in a sequence diagram has nothing in common with the math for laying out an ERD. One-size-fits-all would mean mediocre layout for everything.

## Relations

- [[two-stage-pipeline]] — each type has its own renderer in stage 2
- [[json-schema-contract]] — each type has its own schema defining what fields are available
- [[explainer-mode]] — explainers can embed any of the 10 types inline

## Tags

#archify
