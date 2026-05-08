---
name: fork-sync-conflict
description: Use when resolving a rebase conflict on an HT fork's `ht` branch onto upstream main/master. Triggers from fork-sync-reusable workflow rebase failures, or when manually rebasing an `ht-*/` fork. Encodes the decision recipe and safety rules so an autonomous agent can finish the rebase end-to-end without escalating.
---

# Fork-Sync Conflict Resolution

You are mid-rebase on an HT fork. The mechanical sync (fast-forward `main` from upstream) succeeded but `git rebase main` from `ht` hit a conflict. This skill is the recipe for landing the resolution safely without human escalation.

## The HT fork model (what you're operating inside)

- `main` (or `master` in some forks) is a **pure fast-forward mirror** of upstream — never directly committed to.
- `ht` is the customization branch — HT-specific changes rebased on top of `main`.
- A sync = fast-forward `main` + rebase `ht` onto new `main` + force-push `ht`.
- When upstream changes the same files HT changed, the rebase conflicts. **That's where you come in.**

## Decision flow

### Step 1 — load context before touching anything

```bash
git fetch origin && git fetch upstream 2>/dev/null || git fetch
git status                                          # how many files conflict?
git diff --name-only --diff-filter=U                # which files?
git log --oneline origin/main..origin/ht | head -20 # what HT carries
```

Pull the drift-pattern catalog (categorized recipes per common conflict shape):

```bash
gh api repos/heiervang-technologies/.github/contents/FORK_DRIFT_PATTERNS.md \
  --jq '.content' | base64 -d > /tmp/drift_patterns.md
```

Find this fork's row in `## Per-Fork Drift Profile` — it tells you which drift types are common here.

### Step 2 — categorize each conflict

For every conflicting file, match it to one of the nine drift categories in `/tmp/drift_patterns.md`:

| # | Category | Default action |
|---|----------|----------------|
| 1 | File restructuring (upstream reorganized HT-modified file) | Take upstream's structure, port HT's change into it. Some files have standing rules (e.g. `studio/setup.sh` → always `--theirs`) — check the diary. |
| 2 | Registry / config expansion | Keep both sets of entries; drop dupes if upstream now includes any of HT's. |
| 3 | API surface / signature change | Read upstream's PR; port HT's feature into the new API. Most labor-intensive. |
| 4 | CI / build system | Usually take HT (`--ours`); HT's CI is intentional. |
| 5 | UI / frontend | Re-apply HT's modification on top of upstream's restructured component. |
| 6 | Convergence (upstream absorbed equivalent of HT) | The HT commit is now redundant — `git rebase --skip` if the commit becomes empty after applying upstream. |
| 7 | Duplicate implementation | Adopt upstream's version, remove HT's. If HT's is materially better, leave a note (don't carry both). |
| 8 | Metadata / attribution | Keep upstream's copyright headers. HT should not have modified them. |
| 9 | History rewrite | Only proceed if a `pre-autoresolve-*` or `ht-backup-*` tag exists for rollback. |

### Step 3 — resolve, file by file

```bash
# For each conflict, edit the file, then:
git add <file>

# When all are resolved:
git rebase --continue
```

Loop. Some commits may become empty after upstream absorption — `git rebase --skip` is correct for those (Drift Category 6).

### Step 4 — validate before push (HARD GATE)

All four must pass. If any fail, **stop and exit with an error** — don't push.

```bash
# 1. ht still has HT-specific commits (catastrophic flatten check)
test "$(git rev-list --count origin/main..HEAD)" -gt 0 || { echo "FATAL: ht flattened to main"; exit 1; }

# 2. Pre-autoresolve safety tag exists (rollback point)
git tag -l 'pre-autoresolve-*' | tail -1 | grep -q . || { echo "FATAL: no rollback tag"; exit 1; }

# 3. No leftover conflict markers
! grep -rE '^<<<<<<< |^>>>>>>> |^======= ' . --include='*.py' --include='*.ts' --include='*.tsx' --include='*.rs' --include='*.go' --include='*.c' --include='*.cpp' --include='*.h' --include='*.md' --include='*.yml' --include='*.yaml' --include='*.toml' 2>/dev/null

# 4. Cheap smoke check — package importable / lint clean (best-effort)
[ -f pyproject.toml ] && python -c "import $(basename "$(pwd)" | sed 's/^ht-//; s/[.-]/_/g')" 2>/dev/null
[ -f Cargo.toml ] && cargo check --quiet 2>&1 | tail -5
```

### Step 5 — push

```bash
git push --force-with-lease origin ht
```

Then comment on the open `sync-conflict` issue (if any) with a one-paragraph summary: which drift categories applied, what you did per file, and the new `ht` SHA. Close the issue.

## Hard safety rules — never violate

1. **Never push to `main` or `master` on origin.** The FF-only sync owns that.
2. **Never delete tags** matching `pre-autoresolve-*` or `ht-backup-*`. They're rollback points.
3. **Never use `--force` without `--with-lease`.**
4. **Never auto-close a `sync-conflict` issue** unless validation Step 4 passed end-to-end.
5. **Never modify upstream copyright headers** during conflict resolution.
6. **If you can't resolve confidently, exit non-zero with a clear summary.** The fallback workflow files / updates the tracking issue. Forced wrong answers are worse than a flagged issue.

## When to bail out

Exit early and let the issue-fallback handle it if any of these hold:

- After 3 attempts at `rebase --continue`, the same conflicts keep returning (you're not making progress)
- Conflicts span >10 files in unrelated subsystems (likely a major upstream refactor — needs human judgment)
- Upstream did a directory rename you can't unambiguously map (`git log --diff-filter=R --follow` to confirm)
- You're tempted to delete or modify HT commits whose intent you can't confidently identify from the commit message + diff

A flagged issue with your analysis is more useful than a wrong autoresolve.

## Observability

When you finish (success or bail), produce a short structured summary:

```
fork: <repo-name>
new-ht-sha: <sha>
categories: <comma-separated drift category numbers>
files-resolved: <count>
commits-skipped: <count, if any from Cat 6>
validation: pass | fail (<which gate>)
```

This gets included in the issue comment / workflow summary.
