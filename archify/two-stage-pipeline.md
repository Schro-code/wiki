---
domain: archify
tags: [archify]
sources: []
updated: 2026-10-21
---

# Two-Stage Pipeline

## What it is

The fundamental architecture of archify: a two-stage split where Claude writes a JSON spec (WHAT to show) and a deterministic Node.js renderer produces HTML (HOW to draw it).

## What problem it solves

Claude is excellent at understanding concepts and structuring information, but unreliable at producing pixel-perfect visual layouts. Asking Claude to hand-write SVG with exact coordinates, consistent spacing, themes, and animations would fail constantly. The two-stage pipeline lets each side do what it's good at.

## Notes

```
Request → Claude writes JSON spec → Validator → Renderer (Node.js) → Standalone HTML
```

**Stage 1 — Claude writes JSON.** Claude reads the user's request, understands the concepts, and produces a structured JSON description: components, connections, labels, attributes. Pure meaning, zero pixels.

**Stage 2 — Renderer produces HTML.** A Node.js program reads the JSON and generates pixel-perfect HTML/SVG. It handles geometry (box sizes, arrow routing, spacing), visual design (colors, dark/light themes), and interactivity (animations, export). The renderer is deterministic: same JSON in, same HTML out, every time.

### Diagnosing problems

The pipeline tells you where to look when something goes wrong:

- **Content is wrong** (wrong components, missing connections, mislabeled) — Claude misunderstood the request. Fix by clarifying what you asked for.
- **Layout is wrong** (overlapping boxes, weird arrows) — renderer issue, or Claude wrote valid-but-bad coordinates in the JSON.
- **Total failure** — Claude's JSON didn't match the expected [[json-schema-contract|schema]].

## Relations

- [[json-schema-contract]] — the schema is the contract between the two stages
- [[diagram-type-selection]] — each diagram type has its own renderer in stage 2
- [[explainer-mode]] — the same two-stage pattern applied to full-page documents

## Tags

#archify
