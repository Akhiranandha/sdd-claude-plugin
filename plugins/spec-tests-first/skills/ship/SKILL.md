---
name: ship
description: Phase 5 of the SDD cycle. Use when the user invokes /sdd:ship <feature> to review, commit, push, PR, and merge. Performs an SDD-aware pre-commit code review of the local diff (BLOCK/CAUTION/GO verdict) — checking spec deviations against AC-IDs in spec-status.md, runtime-artifact leaks, weak negative tests, and the standard security / data-loss / concurrency issues. Only on GO does it dispatch commit-commands:commit (no remote) or commit-commands:commit-push-pr (with remote), then runs the marketplace /code-review on the PR diff as a second pass before asking for explicit merge permission.
argument-hint: <feature-name>
allowed-tools: Read, Write, Edit, Glob, Grep, Skill, Bash, AskUserQuestion
---

# /sdd:ship — Phase 5: Review, Commit, PR, Merge

You are running Phase 5 of the SDD cycle for feature **$1**. Inputs: pending changes + `docs/specs/$1/spec.md` + `docs/specs/$1/spec-status.md`.

## Pre-checks

1. **Validation passed?** Read `docs/specs/$1/spec-status.md`. Every `AC-ID` should have status `pass`. If anything is `fail` / `blocked` / `stale`, stop and tell the user to run `/sdd:validate $1` first.

2. **Git initialized and there are pending changes?** If no pending changes, stop with: *"Nothing to ship — no pending changes."*

3. **Git remote configured?** Run `git remote -v`. If there is no remote, ask via AskUserQuestion:

   > No git remote is configured. Choose:
   > - **(a, recommended for solo / sample projects)** Commit locally only — skip push, PR, PR-review, and merge.
   > - **(b)** Configure a remote first (tell me the URL), then proceed with the full ship flow.
   > - **(c)** Cancel.

   On (a): proceed through Step 1 and Step 2 only, then stop after the local commit and report the commit SHA. Skip Steps 3–5.
   On (b): run `git remote add origin <url>`, then continue through all five steps.
   On (c): stop.

---

## Step 1 — SDD-aware pre-commit review (local diff)

Stage all pending changes if not already staged (`git add -A`). Then perform a thorough review against the staged set and end with one of three verdicts: **BLOCK / CAUTION / GO**.

### 1a. Identify scope

Run `git diff --cached --stat` and `git diff --cached`. If the staged set is empty, tell the user — don't fabricate a review.

### 1b. Read full files, not just the diff

For each changed file under ~1000 lines, read the whole file. Diffs alone hide breakage:
- A function call that looks fine in the diff may now reference a removed import.
- A new branch may shadow an existing variable.
- A deleted line may have been the only place that closed a resource or released a lock.

For larger files, read 50–100 lines around each hunk.

### 1c. Read the spec and status

Open `docs/specs/$1/spec.md` and `docs/specs/$1/spec-status.md`. Build two mental sets:

- **Promised AC-IDs** — those marked `pass` in `spec-status.md`. The implementation should match.
- **Promised signals** — for every error-path AC, the discriminating substring / exception class / status named in Section 3 of the spec.

### 1d. Run available local checks

If the project has them, run them and include any failures:

- Linter (`ruff`, `eslint`, `golangci-lint`, etc.)
- Type checker (`mypy`, `tsc --noEmit`).
- Tests for the affected area (`tests/$1/`) — verify they all pass on the staged code.

If running checks would take more than a minute or two, ask first.

### 1e. Review criteria — top-down by severity

Focus your attention top-down. If there are critical issues, do NOT drown them in low-severity nits.

#### Critical (BLOCK)

- **Security** — SQL / shell / template injection, committed secrets / API keys / private keys, auth bypass, missing authorization checks, XSS, SSRF, unsafe deserialization.
- **Data loss** — destructive migrations without backups, unguarded `rm` / `DROP` / `DELETE`, races that can corrupt state.
- **Breaking API / schema changes** without versioning; removed public exports still used externally.
- **Concurrency** — race conditions, lock-ordering bugs, unprotected shared state.
- **Runtime artifacts staged that should be in `.gitignore`** — data files, build outputs, `__pycache__`, `node_modules`, `.env`, IDE configs, lockfile noise. Check the spec's Section 5 for the named runtime data file (e.g. `data.json`) — if it's in the staged set, that's a BLOCK; tell the user to add it to `.gitignore` and re-stage.

#### High (BLOCK unless user explicitly accepts)

- **Spec deviations** — code that doesn't trace to any `AC-ID` marked `pass` in `spec-status.md`. Quote the file:line and ask which AC it serves; if none, ask whether to add an AC or remove the code.
- **Weak negative tests** — tests for an error path that assert only on `exit code != 0` or `status >= 400` without a discriminating substring / exception class / specific status. The SDD test rule forbids these. Quote the test and the spec line that names the signal it should assert on.
- **Promised signals not produced** — an `AC-N.M` says stderr should contain `"X is required"`, but the implementation prints `"validation error"`. The test will fail or pass for the wrong reason.
- **Logic errors** in changed lines — off-by-one, swapped args, mutated inputs, unhandled error paths.
- **Missing error handling** on operations that realistically fail (network, IO, parsing).
- **Incorrect async / await** — missing awaits, fire-and-forget promises, sync code blocking the event loop.
- **Performance regressions** — N+1 queries, accidental quadratic loops, blocking calls in hot paths.

#### Medium (CAUTION — worth fixing, not blocking)

- **Debug code left in** — `print()`, `console.log`, `debugger`, `pdb.set_trace()`, `// TODO: remove` markers in production paths.
- **Code clarity / naming** — confusing names, unclear control flow.
- **Duplication** that should be extracted.
- **Missing or wrong documentation** on public APIs.
- **Patterns inconsistent with surrounding code** — raw SQL when the file uses an ORM; etc.
- **Test names missing the AC-ID anchor** — every SDD test name should embed `AC_N_M` or `AC-N.M`. If a test in the diff lacks it, surface as CAUTION.

#### Low (mention briefly or skip)

- Style nits a formatter would catch.
- Minor refactor suggestions.
- Subjective preferences.

If the diff has any critical or high issues, **omit the low-severity section entirely**. Signal beats noise.

### 1f. Style of feedback

- **Don't restate the diff.** Open with the problem.
- **Quote the line.** `path/to/file.ext:LINE` plus a short snippet so the user can navigate.
- **Explain why, not just what.** "This fails when X because Y" beats "This is wrong."
- **Ask, don't accuse, when intent is unclear.** "Was this intentional? It looks like X, which would normally cause Y."
- **Don't auto-fix.** Report; let the user decide.

### 1g. Output format

```
## Code Review — $1

**Scope:** staged changes (N files)
**Spec context:** `docs/specs/$1/spec.md` (AC-IDs: <list>) — status: <N pass / N stale / N fail>
**Summary:** <X critical, Y high, Z medium, W low>

### Critical
- **path/to/file.ext:LINE** — <one-line problem>
  Why: <one or two sentences>
  Suggested fix: <concrete change>

### High
- ...

### Medium
- ...

### Low
- ... (omit if any critical / high present)

---

**Verdict: BLOCK / CAUTION / GO**
<one-sentence justification>
```

### 1h. Verdict handling

- **BLOCK** → output the issues. Do NOT commit. Tell the user to fix and re-run `/sdd:ship $1`. Stop.
- **CAUTION** → show issues, then use AskUserQuestion: *"Proceed despite cautions? (y/n)"*. Stop on `n`.
- **GO** → continue to Step 2.

### 1i. Edge cases

- **Empty diff** — tell the user, don't fabricate.
- **Generated files** — skip lockfiles, snapshots, minified bundles, generated proto/SDK code. Mention what you skipped in the Summary line.
- **Documentation-only changes** — focus on accuracy and clarity; skip security / concurrency criteria. Verdict should be GO unless docs are misleading.
- **Spec / status / map changes in the diff** — these are expected for SDD; don't flag them. Use them as context.

---

## Step 2 — Commit (and push / PR if remote exists)

If the user chose option (a) in pre-check #3 (no remote), invoke the **`commit-commands:commit`** skill via the Skill tool (commit locally), report the commit SHA, and stop.

Otherwise, invoke the **`commit-commands:commit-push-pr`** skill. Capture the resulting PR URL.

## Step 3 — PR-level review

Invoke the **`/code-review`** slash command from the **`code-review@claude-plugins-official`** plugin against the PR. It runs a multi-agent review (CLAUDE.md adherence, bug scan, git history, prior-PR context, code-comment compliance) and posts findings as PR comments.

Wait for the review to complete. Read the resulting comments from the PR via `gh pr view <PR#> --comments`.

Summarise the verdict to the user:
- If the review posts **no blocking issues** → continue to Step 4.
- If the review posts **blocking issues** → output them, then ask via AskUserQuestion: *"PR review flagged issues. Choose: (a) fix and push another commit, (b) merge anyway, (c) leave PR open."*

## Step 4 — Merge with permission

Use AskUserQuestion:

> Local review: **GO**. PR review: **[summary]**. PR: **[URL]**.
> Merge now? **(y / n)**

- On `y` → merge the PR via `gh pr merge`. Pick the merge style the project uses (squash / merge / rebase) — if uncertain, ask.
- On `n` → leave the PR open and output: *"PR is open at [URL] — merge when you're ready."*

## Step 5 — Post-merge

If merged, output:

> Shipped. Spec `$1` is complete. Status saved in `docs/specs/$1/spec-status.md`.

## Rules

- **Read the spec and status before reviewing.** Without that context you can't detect spec deviations or weak negative tests — the two SDD-specific high-severity classes.
- **Verdict is mandatory.** Step 1 always ends with BLOCK / CAUTION / GO.
- **No commit before GO.** A CAUTION needs explicit user opt-in.
- **No push for option (a) flows.** Local commit and stop.
