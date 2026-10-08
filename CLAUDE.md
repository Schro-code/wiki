# CLAUDE.md — Wiki

Specific instructions for operating within `Wiki/`. Complement the vault root's `CLAUDE.md`.

---

## Session start ritual (mandatory)

At the beginning of any session involving the Wiki, read in this order:
1. `Wiki/index.md` — existing domains
2. The last 5 entries of `Wiki/log.md` — what was done recently
3. The last 3 entries of `Wiki/projects/log.md` — recent project activity
4. `Wiki/tags.md` — available tags
5. `Wiki/labs/index.md` — hands-on learning labs and what concepts they practice
6. `Wiki/projects/index.md` — work project knowledge bases (meetings, notes, decisions)

Do not start any operation without completing this ritual.

---

## Structure

```
Wiki/
├── CLAUDE.md         ← this file
├── PLAN.md           ← system design (reference only, do not modify)
├── index.md          ← root index: list of domains
├── log.md            ← global operations log (append-only)
├── tags.md           ← tag registry: available tags and when to use them
├── raw/              ← raw sources, immutable
│   └── index.md
├── synthesis/        ← pages integrating two or more domains
│   └── index.md
├── labs/             ← hands-on learning labs (one subfolder per lab)
│   └── index.md
├── projects/         ← work project knowledge bases (one subfolder per project)
│   ├── index.md      ← list of active projects
│   ├── log.md        ← project operations log (append-only)
│   └── <project>/    ← one subfolder per work project
│       ├── context.md
│       ├── meetings/
│       └── notes/
└── <domain>/         ← folder per domain (created on demand)
    └── index.md
```

---

## Organization rules

1. **One page per concept** — a single canonical page in its primary domain. No duplicates.
2. **Primary domain** — if a concept crosses domains, it lives in the most representative one. The others link to it with wikilinks.
3. **synthesis/** — only when the value is in the combination, not in an individual concept.
4. **raw/ is immutable** — read, never modify existing files. `raw/index.md` is the only exception: update it when adding a new source.
5. **Domains on demand** — do not create empty folders. Domains emerge from content.
6. **Tags from registry** — only use tags listed in `tags.md`. Never invent a new tag without adding it to the registry first.

---

## Sources in raw/

Every file in `raw/` must have this header at the top:

```
Source: <URL or full reference>
```

If missing, ask the user before ingesting. Without a header, the source is not valid.

---

## Universal page template

```markdown
---
domain: <domain-name>
tags: []
sources: []
updated: YYYY-MM-DD
---

# <Title>

## What it is

<Clear, concise definition. One or two sentences.>

## What problem it solves

<The concrete problem this concept/technology/pattern addresses. Why it exists.>

## Notes

<Free-form body. Technical details, examples, use cases, nuances, comparisons, commands, warnings.>

## Relations

- [[<related-page>]] — <one line explaining the relationship>

## Tags

#<tag>
```

`sources` lists the `raw/` files that fed this page. Values that change over time (versions, dates) go in frontmatter, never in the body.

---

## Ingestion workflow (per source)

1. **Verify header** — confirm URL/reference in the `raw/` file. If missing, ask for it.
2. **Read the source** — read the full file.
3. **Discuss with the user** — key points, questions about focus.
4. **Write or update pages** — create new pages or update existing ones.
5. **Update indexes** — `Wiki/<domain>/index.md` if new pages were created; `Wiki/index.md` if a new domain was created; `Wiki/raw/index.md` if a new source was added to `raw/`.
6. **Mini-lint** — report when done:
   - Pages created and updated
   - Contradictions with existing pages
   - Frequent concepts without their own page (candidates)
7. **Add log entry** — `## [YYYY-MM-DD] ingest | <source title>`

---

## Query workflow

1. Read `Wiki/index.md` — identify relevant domains.
2. Read those domains' `index.md` files.
3. Read the relevant pages.
4. If indexes aren't enough: `grep` across `Wiki/`.
5. Respond with citations to the pages used.
6. If the response synthesized 3+ pages or connected different domains, offer: "Would you like me to archive this response in synthesis/?"

Always review the generated pages (the actual diff), not just the declared intention.

---

## Lint workflow (periodic)

On user request, look for:
- Contradictions between pages
- Stale claims superseded by more recent sources
- Orphaned pages (no incoming wikilinks)
- Concepts repeated across several pages that deserve their own page
- Missing cross-references
- `sources` entries pointing to files that don't exist in `raw/`

Report findings and propose concrete actions.

---

## Projects workflow

Full instructions are in `projects/CLAUDE.md`. Key rules:
- One subfolder per work project, each with `context.md`, `meetings/`, and `notes/`
- Log every operation in `projects/log.md` (append-only)
- Meeting notes are auto-generated by `meetings-agent`

---

## Indexes — format

**`Wiki/index.md`** (root): columns Domain | Description. Update when a new domain is created.

**`Wiki/<domain>/index.md`**: columns Page | When to look | Sources. Update when pages are created or removed. "When to look" describes the situation in which a reader should open that page (e.g. "Look when you want to understand X" or "Look when searching for concepts about Y").

**`Wiki/labs/index.md`**: columns Project | Description | Concepts practiced. Update when a lab is added or removed.

**`Wiki/projects/index.md`**: columns Project | Description | Active. Update when a work project is added or archived.

**`Wiki/projects/log.md`**: append-only log of project operations. Format: `## [YYYY-MM-DD] <type> | <project> | <title>`.

**`Wiki/raw/index.md`**: columns File | Source | Capture date. Update when a new source is added to `raw/`. This is the only file in `raw/` that may be modified.

---

## Log — format

```
## [YYYY-MM-DD] <type> | <title>
```

Types: `ingest`, `query`, `lint`.  
Never edit existing entries. Only append at the end.  
Parseable with: `grep "^## \[" Wiki/log.md | tail -5`
