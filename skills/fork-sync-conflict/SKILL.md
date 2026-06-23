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

For every conflicting file, match it to one of the eleven drift categories in `/tmp/drift_patterns.md`:

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
| 9 | History rewrite | **Autopilot bails — operator territory.** Force-resetting `main` violates rule #1 and requires human judgment about backup-tag freshness. Exit non-zero and let the issue-fallback file a ticket. |
| 10 | Deployment governance / unsanctioned image | Out of scope for the rebase autopilot — this is a registry-side concern (manifest pin vs producible-tag set). Note it in the sync-conflict issue if observed; do not attempt to "resolve" it during rebase. |
| 11 | Silent merge anomalies | Not a per-file conflict — runs as a post-rebase scan. Execute the three-way delta triangulation + survival check from Cat 11. Any `mvU≈0` row or HT-only file touched by a dropped commit goes into the validation report. Do not push if either check is unresolved. |

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

## rerere caveat (read this before you touch a conflict)

`git rerere` ("reuse recorded resolution") looks tempting for HT forks because the same conflicts recur every sync (Drift Cat 1 / 2). **It is unsafe to trust on long-lived divergences and you should default to disabling it.**

Why: rerere caches a resolution keyed by the conflict hunk's content, not by the surrounding structural context. When the surrounding code has changed between conflict-encounters (which it always does on a long-lived fork), rerere can apply a *stale* resolution that drops later commits' changes silently. No conflict markers, no warning — just a working tree that looks plausibly correct but is missing real HT work.

**Near-miss (ht-llama.cpp, 2026-05-12)**: rerere had cached resolutions from exploratory merges on May 3rd. A fresh merge of `origin/ht` (which had since gained LoRA dedupe + auto-discovery in `tools/server/server-models.cpp`) silently applied the May 3rd resolution to that file. The entire LoRA dedupe block, new `common_lora_adapter_info` fields, and the `gguf_is_lora_adapter` signature changes were dropped. Caught only by spot-checking against `origin/ht`.

**Standing rule**:
- Default: `git config rerere.enabled false` before any resolution work on an HT fork.
- If you opt in for a tight-loop standing-rule case (e.g. `studio/setup.sh` in ht-unsloth, where the resolution genuinely is "always take upstream"): first verify the rr-cache is fresh and matches the current structural context. Otherwise nuke it: `mv .git/rr-cache .git/rr-cache.bak.$(date +%Y%m%d)`.
- Never enable rerere in CI automation that runs across many forks with shared cache — there's no shared cache mechanism that's safe across forks anyway, but defense-in-depth.

The autopilot workflow (`fork-conflict-autoresolve-reusable.yml`) sets `rerere.enabled false` by default. Don't override.

## Resolution-time anti-patterns (process hazards)

These are not drift shapes in the upstream — they are *your* failure modes as the resolving agent. All four were caught in real rebases. They feel like shortcuts when you're deep in a conflict; they aren't.

### Soft-reset erasure (looks like a clever flatten, is actually a feature-loss)

Tempting move: `git reset --soft upstream/main && git commit -m "rebase"` — produces a single HT commit on upstream's tip with the right tree. Looks like a fast flatten.

What it actually does: the resulting tree contains HT's files but **silently erases every file that exists only in upstream's `master..main` range**. Tree-identity preserves the HT side; it does not preserve upstream-only paths. You will lose hundreds of upstream-only files with zero conflict markers.

**Rule**: never resolve a divergent rebase via soft-reset. Use `git merge upstream/main` (then resolve conflicts) and *separately* linearize via `git checkout -b ht-linear upstream/main; git checkout ht-merged -- .; git commit` if you want a single commit. The checkout-from-merged step is what preserves upstream-only files.

### Endurance-impaired waivers (you're tired and reaching for `--no-verify`)

After a long resolution session, the temptation to disable a failing check rises: `git commit --no-verify`, `eslint-disable`, `// @ts-ignore`, "we can fix this in a follow-up". These checks usually catch the silent-feature-loss class of bugs — strategy (a) prop-loosening on upstream interfaces that hides a missing HT caller-update.

**Rule**: if you find yourself wanting to bypass a check, that is itself a signal you should bail out and file an issue rather than push. The protocol against waiver-creep is pushback at the call-site, not internal restraint. If the autopilot reaches this point, exit non-zero with a clear summary and let a fresh session pick it up.

### Peer-agent policy contradiction (two voices, one of them wrong)

When coordinating with another agent (e.g. a per-repo dev agent), watch for rule contradictions: one agent says "X is required", the other agent's prior message implied "X is fine". Picking one silently propagates the wrong rule. Surface the conflict.

**Rule**: if two agents (or two prior messages from the same agent) hold contradictory policies, neither agent's local judgment resolves it — escalate to the human (`say` or issue comment) with both positions stated, then wait. Forced consistency without escalation is how silently-wrong rules ship.

### Runbook-hypothesis staleness (your RESUME.md is older than the code)

Resume notes, parked-work READMEs, and issue comments embed *hypotheses about the codebase* at the time of writing. After a rebase advances HEAD, those hypotheses can be wrong — the file you were told to fix may have been renamed, the function signature may have changed, the bug may already be fixed.

**Rule**: before acting on a runbook's named target (file, function, flag), verify it still exists at the named path with the named shape. If not, the runbook is stale — update it before continuing, don't act on the obsolete assumption. A two-line `grep` is cheaper than a wrong fix.

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

## Temporary security watch (while reading the upstream diff)

Until the dedicated security-scan job ships (Phase 5, see `FORK_AUTOMATION_PLAN.md`), this skill carries a best-effort backstop. While you're reading conflict diffs to resolve them, also notice:

- obfuscated payloads (base64 / hex blobs in normally-text files)
- postinstall / preinstall scripts in package manifests fetching external code
- sudden new outbound network calls in build / install / setup scripts
- typosquat-shaped dependency renames
- silent removal of validation / auth / signature checks
- unexpected privilege escalation in install paths

If something looks suspicious — even faintly — handle it like this:

1. **NEVER put exploit specifics in any public-visible artifact.** No PoC, no vulnerable code paths, no repro steps in the issue body. Keep specifics in the workflow run logs (private to the org).
2. Append a `## Security watch` section to the `sync-conflict` issue you create or update, containing ONLY:
   - severity guess: `low` | `medium` | `high` | `critical`
   - upstream commit SHA(s) involved
   - one sentence on the *category* of concern (e.g. "obfuscated payload in postinstall script") — no specifics
   - workflow run URL
3. Mention `@marksverdhei` in the issue body so GitHub emails the owner. This is the temporary notification channel until Phase 5 (irondome + Proton-to-Proton mail) replaces it.
4. If severity is `critical` AND your confidence is high, exit non-zero AFTER filing the issue so the rebased `ht` is not force-pushed. Default `pre-autoresolve-*` tag is the rollback point.

This is a backstop, not a scanner. Don't chase ghosts; flag, don't speculate.

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
