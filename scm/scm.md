---
domain: scm
tags: [scm]
sources: []
updated: 2026-10-14
---

# Software Configuration Management (SCM)

## What it is

The engineering discipline that controls how source code, configurations and artifacts evolve over time. It covers versioning, branching, merging, releasing, and managing the relationship between an organization's codebase and its upstream sources.

## What problem it solves

Software changes continuously and is built by multiple people working in parallel. Without controlled versioning, teams overwrite each other's work, lose track of what changed, ship the wrong version, or cannot reproduce a past state. SCM provides the processes and tooling to keep every change traceable, reversible and reproducible.

## Notes

### Key subdomains

- **Branching strategies** — GitFlow, trunk-based development, feature branches. How teams organize parallel work.
- **Fork management** — Maintaining a customized copy of an upstream project. Patterns: merge-based, rebase-based, layered architecture, vendor branch.
- **Release engineering** — Building, packaging and shipping artifacts from source. Tagging, changelogs, semantic versioning.
- **Patch series management** — Carrying a set of local modifications on top of an upstream codebase (Linux distros, Android device trees).
- **Software supply chain** — Tracking provenance of dependencies, SBOM, license compliance.

### Fork management patterns

| Pattern | How it works | Trade-off |
|---|---|---|
| **Merge-based** | Periodically `git merge upstream/main`. Conflicts only in customized files | Simple; merge commits accumulate |
| **Rebase-based** | Maintain patches on top, rebase on upstream updates | Clean delta; rewrites history, breaks shared branches |
| **Layered architecture** | Customizations in separate files (overrides, config, extensions) that upstream doesn't touch | Zero-conflict syncs; requires upfront design |
| **Vendor branch** | Third-party code on a dedicated branch, merged into main | Clear separation; awkward for deep integrations |

### Key principles

- **Minimize the delta** — The fewer upstream files you edit, the easier every sync.
- **Merge, don't cherry-pick** — Cherry-pick breaks history tracking; merge preserves ancestry.
- **Upstream-first** — Contribute changes back to upstream when possible; every unmerged change is permanent maintenance burden.
- **Layered customization** — Move toward override files, config-driven branching, and extension points so upstream merges produce zero conflicts.

### Key references

- Berczuk & Appleton — *Software Configuration Management Patterns*
- *Pro Git* by Scott Chackon — vendor branches, merge strategies
- Google SRE Book — Release Engineering chapter
- Debian Policy Manual — patch management for distributions
- Linux kernel documentation — maintainer workflow, subsystem trees

## Relations

- [[utopia]] — LDA's fork of deeplethe/utopia is managed using these patterns (merge-based + layered i18n)

## Tags

#scm
