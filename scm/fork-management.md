---
domain: scm
tags: [scm]
sources: []
updated: 2026-10-14
---

# Fork Management

## What it is

The practice of maintaining a customized copy (fork) of an upstream open source project. The fork carries local modifications while periodically absorbing new work from upstream.

## What problem it solves

Organizations adopt open source software and need to customize it — branding, infrastructure, integrations, features the upstream doesn't offer. But the upstream project keeps evolving with bug fixes, security patches, and new features. Without a strategy, every sync becomes a painful manual process where local changes conflict with upstream changes, history gets disconnected, and nobody knows what the fork actually modified.

## Notes

### Vocabulary

- **Upstream** — the original project you forked from.
- **Downstream** — your fork.
- **Delta** — the set of differences between your fork and upstream. Every file you changed, added, or deleted.
- **Carry patches** — same idea as delta, but emphasizes that you must *carry* these differences forward through every sync. The delta is your maintenance burden.
- **Sync** — the act of bringing upstream's new work into your fork.

### The fundamental situation

Upstream evolves. You evolve. Both change the same codebase independently.

```
upstream:  A → B → C → D → E → F        (upstream keeps releasing)
                \
your fork:       C → X → Y → Z           (you diverged at C)
```

X, Y, Z are your carry patches. D, E, F are upstream's new work. The question is always: how do you get D, E, F into your fork without breaking X, Y, Z?

---

### Approach 1: Merge-based

Run `git merge upstream/main`. Git finds the common ancestor (C), compares what each side changed, and combines them.

```
upstream:  A → B → C → D → E → F
                \                 \
your fork:       C → X → Y → Z → M   (M = merge commit)
```

- Files only upstream changed → updated automatically.
- Files only you changed → stay as they are.
- Files both changed → conflict, you resolve manually.

**Pros:** simple, preserves full history, works well with teams.
**Cons:** merge commits accumulate, same files conflict every sync.

**Used by:** most corporate forks, Linux distribution kernels (Ubuntu, RHEL).

---

### Approach 2: Rebase-based

Instead of merging, replay your patches on top of upstream's latest:

```
Before:
upstream:  A → B → C → D → E → F
                \
your fork:       C → X → Y → Z

After rebase:
upstream:  A → B → C → D → E → F
                                  \
your fork:                         X' → Y' → Z'
```

X', Y', Z' are the same changes re-applied on top of F. It looks as if you started your fork today.

**Pros:** your delta is a clean, readable list of commits. Easy to see exactly what the fork customizes.
**Cons:** rewrites history (new commit hashes). Everyone who pulled the old commits must force-pull. Does not work well when multiple people push to the fork independently.

**Used by:** small forks with one maintainer, embedded Linux (Yocto), Android custom ROMs.

#### The multi-person problem with rebase

```
Alice rebases:   C → D → E → F → X' → Y' → Z'
Bob still has:   C → X → Y → Z
```

Alice and Bob now have incompatible histories. Bob must `git pull --force` and discard his local work based on X, Y, Z. This is why rebase is a single-maintainer strategy.

---

### Approach 3: Layered architecture

Not a git strategy — a **design strategy**. The idea: never touch upstream's files. Put customizations in separate files that upstream doesn't know about.

```
Upstream file:       docker-compose.yml
Your customization:  docker-compose.override.yml   ← Docker reads this automatically

Upstream file:       src/i18n/en.ts
Your customization:  src/i18n/custom.ts             ← loaded at runtime as override
```

At sync time, `git merge upstream/main` produces **zero conflicts** because you never edited their files.

**Pros:** syncs are trivial, delta is clearly separated.
**Cons:** requires upfront design. Not always possible for deep changes.

**Used by:** Odoo, WordPress themes, Kubernetes distributions (OpenShift), Android GMS overlays.

---

### Cherry-pick: the anti-pattern

Selectively copying individual upstream commits into your fork with `git cherry-pick`:

```
upstream:  A → B → C → D → E → F
                \
your fork:       C → X → D' → Y → F'     (D' and F' are cherry-picked copies)
```

D' and F' have the same content as D and F but different commit hashes. Git does not know your fork already has those changes. The result:

- `git diff main...upstream/main` shows all upstream commits as "not incorporated", even if you already brought their content.
- Every future merge will re-evaluate those changes and may produce phantom conflicts.
- The more you cherry-pick, the more disconnected the histories become.

**Rule:** merge, don't cherry-pick. Cherry-pick breaks history tracking.

---

### Tactics to reduce the delta

Every file you remove from the delta is one less conflict forever. Ordered from most effective to least:

#### 1. Upstream-first (contribute back)

If your change is useful to everyone, submit it as a PR to upstream. Once merged, it disappears from your delta permanently.

```
You added a CSV export feature.
Upstream would benefit from it too.
→ PR accepted → the feature is now upstream's responsibility, not yours.
```

The best carry patch is one that doesn't exist.

#### 2. Runtime configuration

Instead of hardcoding values into source files, load them from environment variables or a config file.

```
❌  Editing index.html:
    <title>My Company App</title>

✅  Runtime config:
    // config.json (new file, upstream doesn't have it)
    { "title": "My Company App", "logo": "/assets/logo.png" }

    // app.js (upstream's file, untouched)
    fetch('/config.json').then(c => document.title = c.title)
```

The config file is yours. The source file stays upstream's.

#### 3. Composition (separate files combined at build/run time)

Instead of one file that does everything, split into composable layers.

```
❌  Editing docker-compose.yml to add your monitoring stack

✅  Separate files:
    docker-compose.yml              ← upstream's, untouched
    docker-compose.override.yml     ← Docker reads this automatically
    docker-compose.monitoring.yml   ← explicit: docker compose -f ... -f monitoring.yml up
```

Works for anything that supports layered config: Docker Compose, Kubernetes Kustomize, webpack, nginx includes.

#### 4. Override files

Your version is loaded instead of theirs at runtime, without editing the original.

```
❌  Editing upstream's src/i18n/en.ts to add your translations

✅  Override layer:
    src/i18n/en.ts       ← upstream's, untouched
    src/i18n/custom.ts   ← your overrides, merged at runtime
    src/i18n/index.ts    ← loads en.ts, then applies custom.ts on top
```

If `index.ts` already supports this pattern, you only add `custom.ts`. If not, `index.ts` is the one file you edit (and it rarely changes upstream).

#### 5. Build arguments

Same file, different behavior depending on how it's built.

```
❌  Editing the Dockerfile to install monitoring tools

✅  Build arg:
    ARG INSTALL_MONITORING=false
    RUN if [ "$INSTALL_MONITORING" = "true" ]; then \
          apt-get install -y prometheus-node-exporter; \
        fi

    # Your build:
    docker build --build-arg INSTALL_MONITORING=true .
```

If upstream accepts the build arg, your Dockerfile edit becomes zero.

#### 6. Extension points

Register your additions without editing the registration code. This requires upstream to support some form of plugin or hook system.

```
❌  Editing upstream's router to add your route:
    router.route("/custom", post(handler))

✅  Plugin-style registration (if upstream supports it):
    // plugins/my_routes.rs (your file)
    pub fn register(router: Router) -> Router {
        router.route("/custom", post(handler))
    }
```

Many frameworks support this: middleware hooks, plugin registries, event listeners, module auto-discovery.

#### 7. Git attributes (last resort)

For files you *must* edit and can't avoid, tell git to always pick yours:

```
# .gitattributes
docker-compose.yml merge=ours
```

This doesn't eliminate the delta, but eliminates manual conflict resolution — git auto-picks your version for those specific files on every sync.

---

### The tactic ladder (summary)

| Tactic | Effect on delta | Sync conflicts |
|---|---|---|
| Upstream-first | File removed permanently | None |
| Runtime config | File untouched | None |
| Composition | Additions in separate files | None |
| Override files | Original untouched, override loaded at runtime | None (or minimal) |
| Build args | Same file, different behavior | None if upstream accepts the arg |
| Extension points | Registration without editing upstream code | None |
| Git attributes | File still diverges, but auto-resolved | None (auto-resolved) |

### The core principle

> Every upstream file you edit is a permanent conflict point.
> Every file you don't touch syncs for free forever.
> The goal is to move every customization into a file that upstream doesn't know about.

---

### Choosing an approach

| Situation | Recommended approach |
|---|---|
| Team of 2+ people sharing the fork | Merge-based |
| Solo maintainer, small delta | Rebase-based |
| Long-lived fork with frequent syncs | Merge-based + layered architecture |
| Temporary fork (planning to contribute everything back) | Rebase-based + upstream-first |
| Deep customizations across many files | Merge-based + progressive layering |

The ideal long-term state: merge-based sync strategy combined with layered architecture, where `git merge upstream/main` produces zero conflicts because every customization lives in a file upstream doesn't touch.

## Relations

- [[scm]] — parent discipline covering versioning, branching, and release management

## Tags

#scm
