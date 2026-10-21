---
domain: archify
tags: [archify]
sources: []
updated: 2026-10-21
---

# Explainer Mode

## What it is

Archify's second major mode: instead of producing a single diagram, it produces a full structured one-page HTML document with text, panels, and embedded diagrams. It follows the exact same [[two-stage-pipeline|two-stage pattern]] — Claude writes a document spec, the explainer renderer builds the HTML page.

## What problem it solves

Sometimes a single diagram isn't enough to explain something. You need context, reasoning, comparisons, metrics, code snippets — a full explanation. Explainer mode produces a shareable one-pager that combines prose with visuals.

## Notes

### Pipeline comparison

```
Diagrams:   Claude writes JSON spec  →  diagram renderer  →  standalone HTML (one visual)
Explainers: Claude writes document spec  →  explainer renderer  →  standalone HTML (full page)
```

### What an explainer contains

- **Conclusion-first structure** — the answer at the top, reasoning below
- **Structured panels** — sections with typed blocks: prose, lists, callouts, comparisons, facts, metrics, steps, code, diffs, images
- **Native archify diagrams inline** — any of the 10 [[diagram-type-selection|diagram types]] embedded within the page
- **Mermaid charts** — pie, bar, line, sankey, radar, venn for data visualization
- **Dark/light themes** — same as diagrams

### When to use it

The trigger is when explanation, not just visualization, is the goal:

- "Turn this answer into something I can share with the team"
- "Make a page explaining how X works"
- "Create a one-pager about this architecture"

**Heuristic: diagram or explainer?** Am I showing one visual, or explaining something? One visual → diagram. Explanation → explainer.

### Explainer schema

Has its own schema (`explainer.schema.json`) defining: meta, answer (summary/reasons/scope), and panels with typed blocks. The same [[json-schema-contract|schema-as-capability-ceiling]] principle applies.

## Relations

- [[two-stage-pipeline]] — explainer mode is the same two-stage pattern for documents
- [[diagram-type-selection]] — explainers can embed any of the 10 diagram types
- [[json-schema-contract]] — the explainer has its own schema defining what blocks are available

## Tags

#archify
