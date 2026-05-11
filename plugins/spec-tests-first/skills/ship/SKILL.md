---
name: ship
description: Phase 6 of the SDD cycle. Use when the user invokes /sdd:ship <feature> to commit, push, open a PR, and merge. Pre-checks block on outstanding Critical findings from the latest /sdd:review report. Drafts an SDD-aware commit message (AC list, validation summary, review counts) and PR body (Summary, AC checklist, validation checklist, review report link, test plan). Uses inline gh + git — zero external plugin dependencies.
argument-hint: <feature-name>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion
---

# /sdd:ship — Phase 6: Commit, Push, PR, Merge

**Announce at start:** Say to the user: "I'm using /sdd:ship to commit + push + open a PR + (with your permission) merge, with an SDD-aware commit message and PR body. Critical findings outstanding will block — I'll check first." Then proceed.

You are running Phase 6 of the SDD cycle for feature **$1**. Inputs: feature branch with all of `/sdd:build` + `/sdd:fix` + `/sdd:validate` complete. Outputs: a commit (locally if no remote, otherwise commit + push + PR + optional merge).

## Iron Law

> **Ship blocks on outstanding Critical findings. No exceptions, no auto-bypass — if any Critical finding in the latest review report has `Status: pending` or `Status: deferred`, the user must address them (via `/sdd:fix` or explicit override there with justification) before ship proceeds.**

This is the last gate before code goes anywhere outside the developer's machine. The Critical gate exists because a Critical finding in production code is materially worse than one in a feature branch.

## Pre-checks

1. **All ACs pass.** Read `docs/specs/$1/spec-status.md`. Every AC must show `pass`. If any is `fail` / `blocked` / `stale` / `in-progress`, stop:

   > Build is not complete for `$1`. Run `/sdd:build $1` (and `/sdd:validate $1`) first.

2. **No outstanding Critical findings.** Read the report path from `spec-status.md`'s `Latest review:` line. Open the report and grep for finding rows whose header contains `[Critical]`. For each Critical, find the `Status:` line. If ANY Critical has `Status: pending` or `Status: deferred`, stop:

   > Critical findings unresolved in the latest review:
   >   - <id> [Critical] <file:line> — status: <pending|deferred>
   >
   > Address them in `/sdd:fix $1` (or override with explicit justification there) before shipping.

   Findings with `Status: fixed` or `Status: skipped` (with justification recorded) pass this gate.

3. **Validation block exists.** `spec-status.md` should have a `## Validation` block (written by `/sdd:validate`) with no failed VS-N steps. If absent or any VS-N failed, stop and tell the user to run `/sdd:validate $1`.

4. **Pending changes exist?** Run:

   ```bash
   git status --porcelain
   git log --oneline main..HEAD
   ```

   If both are empty (no uncommitted changes AND no unpushed commits ahead of `main`), stop: *"Nothing to ship — feature branch matches main."*

5. **`gh` authenticated?** Run `gh auth status`. If it errors, stop and tell the user to run `gh auth login` first.

6. **Remote configured?** Run `git remote -v`. Record whether a remote exists. If none, AskUserQuestion:

   > No git remote is configured. Choose:
   > - **(a, recommended for solo / sample projects)** Commit locally only — skip push, PR, merge.
   > - **(b)** Configure a remote first (tell me the URL), then proceed with the full ship flow.
   > - **(c)** Cancel.

   On (a): proceed through Step 1 and Step 2 only, stop after the local commit and report the SHA. Skip Steps 3–5.
   On (b): `git remote add origin <url>`, continue through all five steps.
   On (c): stop.

## Phase 6 status update

**After all Pre-checks above pass and before Step 1**, Edit `docs/specs/$1/spec-status.md`'s Phase 6 row: Status = `in-progress`, Updated = today, Notes = `"shipping"`. This lets `/sdd:status` show the in-flight state if the ship flow pauses for user input.

If any Pre-check failed and ship has already aborted, this section never runs — the Phase 6 row stays `pending` so the user knows ship hasn't started.

After a successful merge (or after a local-only commit when there's no remote), update Phase 6 row again: Status = `done`, Notes = `"merged: <PR url>"` (or `"local commit: <short SHA>"` for local-only). If the user leaves the PR open (chose option (e)), set Status = `in-progress`, Notes = `"PR open: <url>"`.


## Step 1 — Stage and gather SDD-aware metadata

```bash
git add -A
git diff --cached --stat
```

If the staged set is empty, there's nothing new to commit beyond what's already on the branch. Skip directly to Step 3 (push) if there are unpushed commits, or stop if neither.

Gather metadata from the SDD artifacts (use Read + Grep, NOT raw bash unless explicitly bash-only):

| Field | Source |
|---|---|
| Feature name | `$1` |
| Spec path | `docs/specs/$1/spec.md` |
| Goal summary | spec.md Section 1 (first 2–3 sentences) |
| AC list | spec.md Section 3 (every `AC-N.M` line) |
| AC pass status | spec-status.md table (every AC = `pass`) |
| VS list | spec.md Section 8 (every `VS-N` line) |
| Validation result | spec-status.md `## Validation` block |
| Review report path | spec-status.md `Latest review:` line |
| Review counts | from the latest report file: count `Status: fixed`, `Status: skipped`, `Status: deferred` for each severity bucket |
| Files touched in this commit | `git diff --cached --name-only` |

Cache these for the commit message + PR body.

## Step 2 — Inline commit with SDD-aware message

### 2a — Detect project commit style

```bash
git log --oneline -20
```

Scan the output. Most messages start with `feat:` / `fix:` / `chore:` / etc. → conventional commits. Otherwise → free-form.

### 2b — Draft the message

For conventional style:

```
feat(<area>): <short summary from spec.md Goal>

Spec: docs/specs/<feature>/spec.md
ACs: AC-1.1 ✓ AC-1.2 ✓ AC-2.1 ✓ AC-2.2 ✓
Validation: <N>/<N> VS passed
Review: <fixed> fixed, <skipped> skipped, <deferred> deferred (0 critical, <high> high outstanding)

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
```

Substitute:
- `<area>` — best-effort area derived from spec path or feature name (e.g. `feat(auth-login): ...` for `feature/auth-login`).
- `<short summary>` — one-line distillation of spec.md Section 1.
- `AC-X.Y ✓` — one per AC, joined by spaces.
- `<N>/<N> VS passed` — total pass count.
- Review counts from Step 1 metadata.

For free-form style, drop the `feat(<area>):` prefix and write a plain title line, then the same body.

### 2c — Confirm with the user

AskUserQuestion (show the full draft message):

> Proposed commit message:
> ---
> <draft>
> ---
> Choose:
> - **(a)** Commit as-is
> - **(b)** Edit message — describe your changes and I'll apply
> - **(c)** Cancel

On (a): commit via HEREDOC:

```bash
git commit -m "$(cat <<'EOF'
<draft message>
EOF
)"
```

Capture the resulting SHA from `git rev-parse HEAD`.

On (b): AskUserQuestion to capture the edited version, then commit.
On (c): stop. The staged changes remain staged for the user to handle manually.

### 2d — Local-only flow exit

If pre-check 6 chose option (a) (no remote), output:

```
Local commit created: <short SHA>
  Spec `<feature>` is implemented and committed locally.
  No remote configured — push, PR, and merge skipped.
```

Stop here. Skip Steps 3–5.

## Step 3 — Push and open PR (remote-only)

### 3a — Push

```bash
git push -u origin "feature/$1"
```

If the push fails (auth error, non-fast-forward, permission denied), surface the error and stop — do not invent a workaround.

### 3b — Draft the PR body

Use the metadata cached in Step 1:

```markdown
## Summary

<2-3 sentences from spec.md Section 1 (Goal)>

## Acceptance Criteria

- [x] AC-1.1: <one-line AC text from spec.md>
- [x] AC-1.2: <one-line AC text>
...

## Validation

- [x] VS-1: <one-line VS text from spec.md>
- [x] VS-2: <one-line VS text>
...

## Code review

- Report: <report path>
- Findings: <fixed> fixed, <skipped> skipped, <deferred> deferred
- Critical outstanding: 0 (gate satisfied)

## Test plan

- [ ] CI passes on the feature branch
- [ ] Reviewer spot-checks the AC ↔ test-name mapping above

🤖 Generated with Claude Code (spec-tests-first plugin)
```

### 3c — Open the PR

```bash
gh pr create --title "<spec-derived title>" --body "$(cat <<'EOF'
<PR body above>
EOF
)"
```

Capture the resulting PR URL from `gh pr view --json url -q .url` or from the create command's stdout.

If `gh pr create` fails (e.g. branch protections require additional checks, no permissions), surface the error and stop. Do not attempt to merge.

## Step 4 — Merge with explicit permission

### 4a — Detect project merge style

```bash
git log --oneline main -20
```

Heuristic:
- Lots of `Merge pull request` commits → merge commit style.
- Many single squashed commits per PR (no merge markers) → squash.
- Linear history with no merges or squashes (rebase + ff) → rebase.

Recommend the matching style as option (a).

### 4b — Ask the user

AskUserQuestion:

> All checks done. PR: <url>.
> Merge now?
> - **(a, matches project style)** Yes — <squash|merge commit|rebase>
> - **(b)** Yes — squash
> - **(c)** Yes — merge commit
> - **(d)** Yes — rebase
> - **(e)** No — leave PR open

### 4c — Execute

Map answer → `gh pr merge` flag:
- squash → `gh pr merge --squash --delete-branch`
- merge commit → `gh pr merge --merge --delete-branch`
- rebase → `gh pr merge --rebase --delete-branch`

Run the command. If merge fails (required checks not passed, conflicts, etc.), surface the error and tell the user the PR remains open at `<url>`.

On (e): output the PR URL and stop:

```
PR is open at <url> — merge when you're ready.
```

## Step 5 — Post-merge

```
Shipped. Spec `$1` is complete.
  Merged: <PR url>
  Status: docs/specs/$1/spec-status.md (all pass)
  Review: <report path> (status preserved)
```

## Red Flags — STOP and reset

| Thought | Reality |
|---|---|
| "I'll skip the Critical-findings pre-check for a small fix" | The pre-check exists for a reason. No exceptions — if the latest review has outstanding Criticals, ship aborts. |
| "I'll merge without asking — the user already approved everything" | The merge step always asks. User-initiated merge is the contract. |
| "Conventional-commits don't fit this project, I'll just write a one-liner" | Detect from `git log --oneline -20`. If the project uses free-form, match free-form. But ALWAYS include the SDD-aware body (Spec / ACs / Validation / Review). |
| "The PR body is verbose — I'll skip the AC checklist" | The AC + VS + Review-summary in the PR body is what makes SDD ships traceable. Don't strip it. |
| "I'll force-push to clean up history" | Never. Use `--no-ff` if needed; never `--force` on feature branches that have been pushed once. |
| "Invoke commit-commands for the commit message" | STF v2 is self-contained. Inline the commit. No external plugin invocations. |

## Rules

- **Always** check Critical findings before staging — outstanding Criticals block the commit.
- **Always** draft an SDD-aware commit message — generic commits waste the available context.
- **Always** ask before committing (the user might want to edit the message).
- **Always** ask before merging — no auto-merge.
- **Never** push to `main` directly. Always feature branch first.
- **Never** `--force`, `--no-verify`, or bypass git hooks unless the user has explicitly asked.
- **Never** invoke external plugins (`commit-commands`, `code-review`, etc.) — STF v2 is self-contained.
- **Never** fabricate a review when none has run. If `Latest review:` is `(none yet)`, stop and tell the user to run `/sdd:review $1` first.
- **System binaries required:** `git` always; `gh` only when a remote is configured.
