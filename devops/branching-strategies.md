---
domain: devops
tags: [devops, scm]
sources: []
updated: 2026-10-16
---

# Branching Strategies

## What it is

A branching strategy is the convention a team adopts for organizing work across Git branches: which branches are long-lived, when to branch, when to merge, and how code flows toward production. The strategy determines the pace of integration and the shape of your CI/CD pipeline.

## What problem it solves

Without an agreed convention, teams make arbitrary branching decisions. Some branches diverge for weeks, producing massive merges. Others push directly to main and break everyone. A strategy sets clear rules so parallel work stays integrable and the path to production is predictable.

## Notes

### The three dominant strategies

#### 1. Gitflow (Vincent Driessen, 2010)

The model you described is closest to a simplified Gitflow.

**Long-lived branches:**
- `main` — production-ready code at all times
- `develop` — integration branch where features accumulate

**Short-lived branches:**
- `feature/<name>` — one branch per feature, branched from `develop`
- `release/<version>` — stabilization branch; branched from `develop`, merged into `main` + `develop`
- `hotfix/<name>` — emergency fix; branched from `main`, merged into `main` + `develop`

**Flow:**
```
feature/* ─→ develop ─→ release/* ─→ main (tag)
hotfix/*  ─────────────────────────→ main (tag) + develop
```

**When to use:** Projects with versioned releases (libraries, mobile apps, products with discrete version numbers) where you need to maintain multiple versions in parallel.

**Trade-off:** Complex. The `develop` branch becomes a bottleneck. Features can sit unreleased for weeks. The double-merge into `main` + `develop` is error-prone.

---

#### 2. GitHub Flow (simplified, continuous delivery)

Only one long-lived branch: `main`. Everything else is short-lived.

**Flow:**
```
main ─→ feature/* ─→ PR ─→ review ─→ merge ─→ main ─→ deploy
```

Rules:
1. `main` is **always** deployable
2. Create a branch for any work — feature, fix, experiment
3. Open a PR when ready for review
4. Deploy to staging from the branch before merging
5. Merge to `main` → triggers automatic deploy to production

**When to use:** Web services that deploy continuously, small-to-medium teams, products without discrete version numbers.

**Trade-off:** Requires strong CI/CD. Undisciplined teams break `main`. No natural place to accumulate a release.

---

#### 3. Trunk-Based Development (TBD)

The most extreme form of continuous integration. Everyone commits to a single shared trunk (`main`) directly, or via branches that live no longer than 1–2 days.

**Flow:**
```
main ← [tiny commit or 1-day branch] ← developer A
main ← [tiny commit or 1-day branch] ← developer B
```

Unfinished features are hidden behind **feature flags** (runtime switches), not behind long-lived branches.

**When to use:** High-trust, highly automated teams. Google, Facebook, and other elite performers use this. Requires near-100% test coverage and strong feature flag infrastructure.

**Trade-off:** Demands discipline. Any bad commit immediately affects everyone. Feature flags add code complexity and cleanup debt.

---

### Comparison

| | Gitflow | GitHub Flow | Trunk-Based |
|---|---|---|---|
| Long-lived branches | `main`, `develop` | `main` only | `main` only |
| Feature isolation | Long-lived feature branches | Short-lived PRs | Feature flags (not branches) |
| Release rhythm | Batched releases | Continuous | Continuous |
| Merge complexity | High (double merges) | Low | Lowest |
| CI/CD requirement | Medium | High | Very high |
| Rollback mechanism | Revert or hotfix branch | Revert commit | Revert commit or flag off |

### Your model in context

Your current mental model — `dev` branch, `main` branch (prod), and `feature/*` branches that PR into `dev` — is a common simplified variant of Gitflow without `release/*` or `hotfix/*` branches. It sits between Gitflow and GitHub Flow:

- You get an integration buffer (`dev`) before production
- You avoid the complexity of release branches
- Trade-off: `dev` can accumulate stale code if features pile up without being promoted to `main`

The key decision your model forces: **how often does `dev` → `main` happen?** If rarely, you have big-bang releases with high risk. If frequently (every sprint or on demand after passing staging), you get most of the benefits of GitHub Flow.

## Relations

- [[devops-overview]] — branching strategies are one of the core Flow levers
- [[ci-cd-pipeline]] — the strategy determines which branch triggers which pipeline stage
- [[scm]] — SCM subdomain; fork management uses different branching conventions

## Tags

#devops #scm
