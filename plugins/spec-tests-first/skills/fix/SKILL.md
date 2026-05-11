---
name: fix
description: Phase 4 of the SDD cycle. Use when the user invokes /sdd:fix <feature> to walk through findings in the latest /sdd:review report. Reads ./reports/code-review_<feature>_*.md (resolved via spec-status.md's `Latest review:` pointer), iterates findings severity-then-file, asks the user (fix / custom / skip / defer / quit) per finding, applies the edit, dispatches test-runner to re-check the feature's tests, reverts that single edit if any AC regresses, mutates the report's `Status:` field in place, atomic-commits each successful fix, gates on Critical findings before allowing /sdd:validate, and (recommended) re-runs /sdd:review at the end. Resumable — re-invoking picks up at the first finding still marked `pending`.
argument-hint: <feature-name> [--report <path>]
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion, Agent, Skill
---

# /sdd:fix — Phase 4: Walk Findings, Fix or Defer

**Announce at start:** Say to the user: "I'm using /sdd:fix to walk findings from the latest review one-by-one, with revert-on-regression and atomic commits per successful fix. Critical findings must be addressed before /sdd:validate." Then proceed.

You are running Phase 4 of the SDD cycle for feature **$1**. Inputs: the latest review report + feature source on `feature/$1`. Outputs: applied edits + atomic commits per successful fix + mutated report file (per-finding `Status:` updated, `## Fix log` appended).

## Iron Law

> **After every edit, the test-runner runs against the full feature suite. If any previously-pass AC now fails, the edit is reverted (single `git checkout`), the finding is marked `deferred`, and we move on. Never commit a fix that breaks an AC.**

The whole reason we walk findings one-by-one (rather than batch-applying) is so this revert is atomic — one finding, one diff, one rollback. The user sees the regression message and decides; we never silently accept.

## Pre-checks

1. **Spec status complete?** Read `docs/specs/$1/spec-status.md`. Every AC must be `pass`. Otherwise stop:

   > Build is not complete for `$1`. Run `/sdd:build $1` first.

2. **Resolve the report path.**
   - If invoked with `--report <path>`, use that.
   - Else read `spec-status.md`'s `Latest review:` line and use that path.
   - Else glob `./reports/code-review_$1_*.md` (sorted by mtime descending) and pick the newest.
   - If none of the three resolve → stop:

     > No review report found for `$1`. Run `/sdd:review $1` first.

3. **Working tree state.** Run `git status --porcelain`. If dirty, stash:

   ```bash
   git stash push -u -m "sdd:fix pre-start stash for $1"
   ```

   Tell the user the stash was created. Fixes will land on the feature branch as atomic commits; the stash protects unrelated WIP.

4. **Test command + path resolution.** Read `CLAUDE.md ## Test commands` + `## Test layout` (multi-service-aware — see `/sdd:build` Step 2/3 for parsing). Cache the resolved per-service command(s) and `tests_root`(s) for use in Step 4's regression check.

## Step 1 — Parse the report

Read the report file fully. For each finding, extract these fields by parsing the standard reporter format:

| field | how to extract |
|---|---|
| `id` | the leading `[SEC-NNN]` or `[QUA-NNN]` token in the finding's header line |
| `severity` | the bracketed severity in the header (`[Critical]`, `[High]`, `[Medium]`, `[Low]`, `[Informational]`) |
| `title` | one-line problem statement (the `Issue:` line, or the header trailer after the bracketed tokens) |
| `file:line` | the backtick-quoted `<file>:<line>` in the header |
| `why` | the `Why it matters here:` body line |
| `suggested_fix` | the `Remediation:` body line |
| `status` | the `Status: <value>` line at the end of the finding |

Findings already marked `Status: fixed` or `Status: skipped` are **not re-prompted** in Step 3. Findings marked `Status: deferred` from a prior `/sdd:fix` run ARE re-prompted (they got past once with revert-on-regression or user opt; user gets another chance to act).

If the report has zero findings (or zero `pending`/`deferred` findings), short-circuit:

> All findings in `<report path>` are already addressed (or none exist). Nothing to fix.
> Next: `/sdd:validate $1`.

Stop.

## Step 2 — Build the work queue

Sort the queue:

1. By severity descending: Critical → High → Medium → Low → Informational.
2. Within each severity bucket, by file (alphabetical), so edits to one file are clustered.

Skip findings whose current `Status` is `fixed` or `skipped`. Include `pending` and `deferred`.

Print a brief queue summary before walking:

```
Walking <N> findings in `<report path>`:
  Critical: <c>   High: <h>   Medium: <m>   Low: <l>   Informational: <i>
  Already addressed (skipped/fixed): <s>
```

## Step 3 — Walk the queue, one finding at a time

For each pending finding, show it and ask the user:

```
[<id>] [<severity>]  <file>:<line>
Title: <title>
Why:   <why>
Suggested fix: <suggested_fix>
```

Use AskUserQuestion:

> Action for `<id>`?
> - **(f) Fix** — apply the suggested fix; I'll run tests after and revert if any AC breaks
> - **(c) Custom** — describe a different fix and I'll apply your version
> - **(s) Skip** — mark fixed without editing (already addressed elsewhere, or false positive)
> - **(d) Defer** — leave as pending, revisit later (next `/sdd:fix` run will re-prompt)
> - **(q) Quit** — keep progress so far and stop the loop

### Critical-severity Skip is gated

If `severity == Critical` and the user picks `(s) Skip`, follow up with another AskUserQuestion:

> Marking a Critical finding as skipped without editing. Justification?
> - **(a) False positive** — explain why
> - **(b) Mitigated elsewhere** — note where
> - **(c) Accepted risk** — note the rationale
> - **(d) Cancel skip** — return to the previous prompt

Capture the user's one-line justification verbatim. Record it in the `## Fix log` row's `note` column. The Critical gate in Step 6 will see this finding as `skipped` and accept it, but the reason is in the audit trail.

## Step 4 — Apply with revert-on-regression

When the user picks `(f) Fix` or `(c) Custom`:

### 4a — Apply the edit

For `(f)`: apply the `suggested_fix` from the finding. Use the Edit tool, scoped to the file in the finding's `file:line` (and strictly related files if the fix logically spans them — e.g. extracting a helper into a new module).

For `(c)`: AskUserQuestion to capture the user's edit description (or have them point at the file/line and describe the change inline), then apply it.

### 4b — Regression check

Dispatch the `test-runner` subagent against the feature's tests with the FULL AC-ID list from `spec-status.md` (multi-service: dispatch once per service, accumulate results):

```
Agent[test-runner]
  subagent_type: test-runner
  prompt: |
    test_command: <resolved command for this service>
    test_path: <tests_root for this service, scoped to this feature>
    expected_ac_ids: [<full AC list>]
```

Parse the returned JSON (first ```json fenced block).

### 4c — Evaluate

- **All previously-`pass` ACs still pass** (every AC in `passed`, none in `failed` or `errored`):
  1. Update the finding's `Status:` line in the report to `Status: fixed` (in-place Edit).
  2. Atomic commit:
     ```bash
     git add -A
     git commit -m "fix(sdd:review): <finding-id> — <one-line title>"
     ```
  3. Append a row to the `## Fix log` table (see Step 5):
     `<ISO timestamp> | <id> | fix | success | <SHA> | <suggested_fix one-liner or 'custom: <user note>'>`
  4. Move to the next finding.

- **Previously-`pass` AC now `failed` or `errored`** (regression):
  1. Revert just this edit:
     ```bash
     git checkout -- <file(s) touched by the edit>
     ```
     (If the edit was a new file creation, `git clean -f <new file>` or `rm <new file>` — use `git status` to determine.)
  2. Update the finding's `Status:` line to `Status: deferred`.
  3. Append to `## Fix log`:
     `<ISO timestamp> | <id> | fix | reverted | - | broke AC-<N.M>, reverted`
  4. Show the user the regression message (the AC and failure detail from the test-runner JSON).
  5. AskUserQuestion: **Continue with next finding** or **Quit**? On Quit → fall through to Step 6 with the current queue state.

- **test-runner errored** (`runner_failed` or `json_parse_failure`):
  1. Surface `stdout_tail` from the JSON.
  2. Update the finding's `Status:` line to `Status: deferred`.
  3. Append to `## Fix log`:
     `<ISO timestamp> | <id> | fix | error | - | test-runner error: <short reason>`
  4. AskUserQuestion: continue / quit?

### 4d — Skip / Defer / Quit (no edit applied)

- `(s) Skip`:
  - Status line → `Status: skipped`.
  - Fix log row: `<ts> | <id> | skip | - | - | <reason or "(no justification)">`. Critical-severity skips MUST include the Step 3 justification verbatim.
  - No commit (nothing changed in code).

- `(d) Defer`:
  - Status line → `Status: deferred`.
  - Fix log row: `<ts> | <id> | defer | - | - | -`.
  - No commit.

- `(q) Quit`:
  - Stop walking the queue.
  - Fall through to Step 6 (Critical gate evaluation) and Step 7 (re-review prompt) based on current state.

## Step 5 — Mutate the report file

Two kinds of mutation, both via the Edit tool:

### 5a — Per-finding `Status:` updates

Use Edit with a unique anchor. The reporter's format puts `Status: pending` as a line on its own, so:

```
old_string: "Status: pending"   (within the finding's block — make it unique by including the surrounding lines)
new_string: "Status: fixed"     (or skipped / deferred)
```

If `Status: pending` is not unique within the file (multiple findings still pending), include the finding's `id` in the anchor:

```
old_string: |
  - **[SEC-001] [Critical]
  ...
  Status: pending
new_string: |
  - **[SEC-001] [Critical]
  ...
  Status: fixed
```

### 5b — Append to `## Fix log`

Append at the END of the report file. If `## Fix log` already exists (from a prior `/sdd:fix` run), append rows to its table. If not, append a fresh section:

```markdown

## Fix log

| time                | finding | action | result   | commit  | note                 |
|---------------------|---------|--------|----------|---------|----------------------|
| 2026-05-11 14:23:01 | SEC-001 | fix    | success  | abc1234 | parameterized query  |
```

Use ISO-8601-ish timestamps (`date +"%Y-%m-%d %H:%M:%S"` or equivalent — Python / Node fallbacks per the reporter's pattern). Use the 7-char short SHA from `git rev-parse --short HEAD` after the atomic commit.

Re-runs of `/sdd:fix` add more rows, never rewrite history.

## Step 6 — Critical gate + Phase 4 status update

### 6a — Mark Phase 4 = `in-progress` when the walk starts

At the very beginning of Step 3 (before the first finding is shown), Edit `docs/specs/$1/spec-status.md`'s Phase 4 row: Status = `in-progress`, Updated = today, Notes = `"walking <N> findings"`.

### 6b — Critical gate

After the queue is exhausted (or user quit), evaluate:

- Walk every finding in the report (re-parse to get fresh status values).
- Count `Status: pending` and `Status: deferred` entries with `severity == Critical`.

If **any Critical is `pending` or `deferred`**, stop with:

```
Critical findings unresolved for `$1`:
  - <id> [Critical] <file:line> — <title> — current status: <pending|deferred>
  - <id> [Critical] <file:line> — <title> — current status: <pending|deferred>

Address them before /sdd:validate. Options:
  (a) Re-run /sdd:fix $1 and walk only the deferred Criticals
  (b) Override with explicit reason — I'll mark them skipped with your justification recorded
  (c) Stop here
```

AskUserQuestion. On (b): for each unresolved Critical, capture a justification line and mark `Status: skipped` with the reason in the Fix log. On (a) or (c): stop the phase; user is expected to re-run.

If all Criticals are `fixed` or `skipped` (with reasons), continue to Step 7.

## Step 7 — Optional re-run of `/sdd:review` (recommended)

AskUserQuestion:

> Re-run `/sdd:review $1` on the fixed code to confirm a clean state?
> - **(a, recommended) Yes** — produce a fresh report; if clean, continue to `/sdd:validate`
> - **(b) No** — trust the per-fix regression checks; continue to `/sdd:validate` now
> - **(c) Stop here** — I'll inspect the diff first

On (a):
1. Invoke the `review` skill from this plugin: `Skill(skill="review", args="$1")`.
2. The review writes a new timestamped report and updates `spec-status.md`'s `Latest review:` pointer.
3. Inspect the new report:
   - Any Critical findings → return to Step 1 of this skill on the NEW report (recurse — the user can quit at any time).
   - No Criticals → output the next-step pointer and stop.

On (b): output the next-step pointer.
On (c): stop, leaving state preserved.

### 7a — Mark Phase 4 = `done` on success

Before printing the next-step pointer, Edit `spec-status.md`'s Phase 4 row: Status = `done`, Updated = today, Notes = `"<fixed> fixed, <skipped> skipped, <deferred> deferred (0 critical outstanding)"`.

If the user quit early with Critical findings still unresolved (Step 6 gate failed and they chose "stop here"), leave Phase 4 = `in-progress` with Notes = `"<N> critical unresolved — paused"`. Don't promote to `done` — the gate isn't satisfied.

### Next-step pointer on success

```
Fix complete for `$1`.
  Report: <report path>
  Findings: <fixed> fixed, <skipped> skipped, <deferred> deferred (0 critical outstanding)

Next: /sdd:validate $1
```

## Edge cases

| Case | Behavior |
|---|---|
| No report exists | Stop in pre-check 2. "Run `/sdd:review $1` first." |
| Multiple reports for the same feature | Always use the newest (from `Latest review:` or glob mtime). User can override via `--report <path>`. |
| Suggested fix references code that has drifted | Edit will fail (`old_string` not unique / not found). Mark `Status: deferred`, note "code drifted; re-run `/sdd:review` for fresh context". |
| User quits in the middle | Report is already mutated for findings processed so far. Re-invoking picks up where left off. |
| Atomic commit fails (e.g. pre-commit hook rejects) | Surface the hook output. Do NOT update the finding's `Status:` (the edit is on disk but uncommitted). User decides: fix the hook issue or revert the edit. |
| `Latest review:` points at a path that no longer exists on disk | Fall back to glob. If no match, stop. |
| Report mutates while `/sdd:fix` is running (concurrent edit) | Out of scope — single-user flow. Don't guard against this. |

## Resumability

The report file is the single source of truth and is mutated in place. Calling `/sdd:fix $1` a second time picks up exactly where it left off — no state lives in memory between sessions. The `## Fix log` provides the audit trail across multiple sessions.

## Red Flags — STOP and reset

| Thought | Reality |
|---|---|
| "This fix is small, surely it won't regress" | Run test-runner anyway. The cost is 30 seconds; the cost of a silent regression is much higher. |
| "I'll batch a few similar fixes, then test once" | One edit at a time. Batching destroys revert atomicity — if any AC fails, you can't tell which fix broke it. |
| "The user said skip — I'll just mark it fixed" | `Skip` = "don't edit, mark as `skipped`". For Critical findings, `Skip` requires a one-line justification recorded in the report. No silent skips. |
| "Critical gate is annoying — let me proceed to validate" | The Critical gate is the whole reason this phase exists. Override needs explicit user reason; no auto-bypass. |
| "Re-running /sdd:review at the end is wasteful" | It's recommended for a reason — fixes can introduce new findings (extracted helper that's now dead, magic-number-replaced constant with a typo). The 30-60s cost is worth it. |
| "The fix touches multiple files — I'll commit the AC test edit separately" | Tests aren't part of the fix. If a fix needs to touch tests, the test is asserting on implementation details (not behavior) — flag it and ask the user. |

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Atomic commits clutter history" | Atomic commits make revert cheap and audit clean. The clutter argument loses against the safety win. |
| "I'll defer all Lows to save time" | Defer is fine — but it's a user choice, not yours. Show the finding and let them pick. |
| "Custom fixes are slower than just applying the suggestion" | Sometimes the suggestion is wrong for the project's idioms. The user knows their code; trust their `(c) Custom` choice. |
| "The report says `Status: pending` but I already fixed it elsewhere" | Use `(s) Skip` with justification = "already fixed in <commit/diff>". Don't silently flip `Status:` without a Fix log row. |

## Rules

- **Always** dispatch `test-runner` after each `(f)` or `(c)` edit, before committing. No exceptions.
- **Always** atomic-commit each successful fix. No batching. Clean history is the contract.
- **Always** revert a single edit on regression — never try to "fix the fix". Mark `deferred` and move on.
- **Always** mutate the report in place (per-finding `Status:` line) — no shadow copy, no "fix.md" derivative.
- **Never** weaken or modify the AC tests to make a fix pass. Fix code, not tests.
- **Never** auto-fix multiple findings in one shot. One at a time, with the user's explicit per-finding choice.
- **Never** push to remote. That's `/sdd:ship`'s job.
- **Never** invent finding IDs, severities, or remediations. Everything in the work queue comes from the report.
