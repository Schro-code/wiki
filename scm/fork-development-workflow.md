---
domain: scm
tags: [scm]
sources: []
updated: 2026-10-15
---

# Fork Development Workflow

## What it is

A practical workflow for fixing bugs and developing features on a fork that must stay synchronized with upstream. Covers the full cycle: tracking what you changed, fixing bugs that affect both codebases, testing before pushing, and keeping the upstream PR clean.

## What problem it solves

When maintaining a fork, a bug fix often touches upstream files. Without a method, you lose track of which files were modified (creating surprise conflicts at sync time), you push broken code to upstream because you didn't test against their CI requirements, and you end up with messy PRs full of trial-and-error commits that reviewers reject.

## Notes

### The tracking system

Two files track every edit to upstream files:

| File | Format | Purpose |
|---|---|---|
| `carry-patches.json` | JSON array | Machine-readable inventory for the drift detection pipeline. Each entry: `id`, `file`, `description` (what AND why), `lines` |
| `docs/design/fork-sync-guide.md` | Markdown table | Human-readable reference. Same info, different audience |

Both must stay synchronized. If one has a patch the other doesn't, something went wrong.

**When you edit an upstream file**, update both in the same PR. When you remove a carry patch (upstream absorbed it, or you moved the logic to a fork-only file), delete from both.

### The dual-PR workflow

When a bug requires editing an upstream file, you produce two PRs:

```
1. Fix on fork branch          → PR to your repo's main
   - lib.rs change
   - carry-patches.json update
   - fork-sync-guide.md update

2. Fix on upstream PR branch   → PR to upstream's dev
   - lib.rs change (same logic)
   - tests (wiremock, unit)
   - no fork-specific files
```

The fork PR carries the patch tracking. The upstream PR carries the tests. If upstream accepts the PR, you delete the carry-patch entry — one less conflict forever.

### The workflow step by step

#### 1. Diagnose the problem

Identify which file needs changing and whether it's an upstream file. If it is, you'll need both PRs.

#### 2. Fix and test on the fork first

Create a branch from your fork's `main`. Make the minimal edit. Rebuild (`make up` or equivalent). Test against the real endpoint — don't assume it works because the code looks right.

**Lesson learned:** API error formats vary between providers. OpenAI returns `{"error": {"message": "..."}}` (nested). Azure OpenAI can return `{"message": "..."}` (flat). If your detection checks only one format, it silently fails on the other. Use the existing error-parsing helpers (`err_detail()` in this codebase) that handle all formats.

#### 3. Commit on the fork branch

Include the `lib.rs` change, `carry-patches.json` update, and `fork-sync-guide.md` update in one commit. This is your fork PR.

#### 4. Create the upstream PR branch

Start from `upstream/dev`. Apply the same logic to `lib.rs`. Add tests — the upstream project requires them. Do NOT include fork-specific files.

#### 5. Run the upstream CI checklist locally

Before pushing to the upstream PR:

```bash
cargo fmt --check -p <crate>       # formatting
cargo clippy -p <crate> -- -D warnings  # linting
cargo test -p <crate>              # tests pass
```

All three must pass. Don't push and wait for CI to tell you — run them locally first.

#### 6. Squash before pushing

If you iterated (fix → broken CI → fix again), squash into one clean commit before pushing:

```bash
git reset --soft upstream/dev
git add <file>
git commit -s -m "One clean commit message"
git push --force-with-lease
```

Reviewers see one commit, not your trial-and-error history.

#### 7. Respond to the reviewer

Address each point they raised. Quote the code that fixes it. Mention what you verified locally (`cargo fmt`, `cargo clippy`, `cargo test` — how many passed).

### Key lessons

| Lesson | Detail |
|---|---|
| Test against the real endpoint | Code that looks correct can fail silently if the error format doesn't match your detection |
| Use existing helpers | `err_detail()` handles multiple error formats. Don't reinvent the detection |
| Run CI locally before pushing | `cargo fmt`, `cargo clippy -D warnings`, `cargo test`. Every time, no exceptions |
| Squash messy history | One clean commit per PR. Force-push is fine on your own PR branch |
| Fork PR ≠ upstream PR | Fork carries patch tracking files. Upstream carries tests. Don't mix them |
| Fewer changes = better | The reviewer said "change the least possible." Every line you add is a line to maintain |

## Relations

- [[fork-management]] — the theory: sync strategies, layered architecture, tactics to reduce delta
- [[scm]] — parent discipline

## Tags

#scm
