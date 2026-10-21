---
domain: archify
tags: [archify]
sources: []
updated: 2026-10-21
---

# JSON Schema Contract

## What it is

A set of 12 JSON schemas (one per diagram type, one for explainers, one shared `common.schema.json`) that define exactly what valid JSON looks like for each diagram type. The schema is the machine-readable contract between Claude and the renderer.

## What problem it solves

Without a contract, Claude would have to guess what fields the renderer expects, leading to silent failures or broken output. The schemas make the interface explicit, validated, and discoverable.

## Notes

Each schema says precisely what fields exist, which are required, which are optional, and what values are valid. For example, a workflow schema requires `lanes`, `nodes`, `edges`, and each node must have an `id`, a `label`, a `col` between 0-5.

### Three things the schema gives you

1. **Validation** — before the renderer runs, a validator checks Claude's JSON against the schema. Bad JSON gets caught early with a clear error message, not a broken diagram.
2. **Guidance for Claude** — the schema tells Claude exactly what fields exist and what's possible. Claude can look up the schema to know what to write.
3. **Capability ceiling** — if a field isn't in the schema, archify can't do it. The schema IS the answer to "can archify do X?"

The third point is the most useful for intuition: **the schema defines what's possible for each diagram type.** If you want to know whether a feature exists, check the schema.

### Schema locations

Schemas live at `~/.claude/skills/archify/schemas/`. The shared definitions (`common.schema.json`) include reusable types: icons (22 values), brand marks, visual presets, animation settings, locale/translations.

## Relations

- [[two-stage-pipeline]] — the schema connects the two stages
- [[diagram-type-selection]] — each of the 10 types has its own schema with different fields

## Tags

#archify
