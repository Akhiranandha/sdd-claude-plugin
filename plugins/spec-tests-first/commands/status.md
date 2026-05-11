---
name: status
description: Show SDD progress across all specs in the current project, or drill into one feature's per-AC status. Reads docs/specs/*/spec-status.md and renders a pass/fail/blocked/stale summary table. Read-only — never modifies files.
argument-hint: "[<feature>]"
allowed-tools: Read, Glob, Bash
---

# /spec-tests-first:status — SDD progress summary

You are running `/spec-tests-first:status` against the user's current project (NOT the plugin directory). Output: a single markdown table plus one summary line. This command is **read-only** — do not write files, do not run tests, do not propose next actions beyond the summary line.

## Argument

`$1` — optional feature name.

- **No arg → aggregate mode**: one row per spec across the whole project.
- **`<feature>` arg → drill-down mode**: one row per AC-ID inside that feature's spec.

## Step 1 — Locate spec-status.md files

Use Glob, relative to the user's current working directory:

- Aggregate mode: pattern `docs/specs/*/spec-status.md`
- Drill-down mode: pattern `docs/specs/$1/spec-status.md`

If the glob returns nothing:

- Aggregate mode: print exactly `No specs found under docs/specs/. Run /spec-tests-first:spec <feature> to create one.` and stop.
- Drill-down mode: print exactly `No spec-status.md for feature "$1". Either the feature doesn't exist or /spec-tests-first:build hasn't run yet.` and stop.

## Step 2 — Parse each spec-status.md

For every matched file:

1. **Feature name** = the directory name directly under `docs/specs/`.
2. **AC entries**: scan the file line-by-line and match `AC-N.M` (regex `AC-\d+\.\d+`) followed — anywhere in the same line, after any separator characters — by one of the status keywords: `pass`, `fail`, `blocked`, `stale`, `not-started`, `removed`, `in-progress`. Match case-insensitively.
   - Be format-tolerant: markdown tables, bullet lists, YAML, or plain prose are all fine. You're matching `(AC-ID, status)` pairs, not parsing a specific structure.
   - If the same AC-ID appears multiple times, the **last** occurrence wins.
   - Lines that contain an AC-ID but no recognized status keyword: skip silently.
3. **Last updated**: the file's modification time. Get it with Bash. On Windows use `powershell -Command "(Get-Item '<path>').LastWriteTime.ToString('yyyy-MM-dd')"`. On POSIX use `stat -c %y '<path>' | cut -d' ' -f1`. If neither works, write `unknown`.
4. **Notes** (drill-down only): the rest of the matched line after the status keyword, trimmed. Strip leading separators (`|`, `-`, `:`, whitespace). Empty string if there's nothing left.
5. **Latest review** (v2 addition): scan for a line matching `^Latest review:\s*(.+)$` at the top of the file. If present, capture the value (a path or `(none yet)`). If absent (older spec-status.md from v1), record `n/a`.
6. **Findings rollup** (v2 addition): if `Latest review` points at a real file, open it and grep for `Status:` lines plus their associated severity bracket. Count `Status: pending` and `Status: deferred` per severity bucket. Track:
   - `findings_total` — total `Status:` lines in the report.
   - `findings_fixed` — count of `Status: fixed`.
   - `critical_outstanding` — count of `Status: pending` + `Status: deferred` with `[Critical]` in the same finding block.
   - If the report file is missing or unreadable, set all three to `?`.
7. **Current phase** (v2.1 addition): scan for the `## Phase progress` table. The "current phase" is the lowest-numbered row whose Status is NOT `done` AND NOT `pending` (i.e. the active row — `in-progress`, `fail`, or `blocked`). Format as `<N>. <name> (<status>)`. If all rows are `done`, current = `complete`. If all rows are `pending` (build hasn't started), current = `not-started`. If the `## Phase progress` table is absent (older v1 spec-status.md), current = `n/a`.

If a file exists but contains zero matched AC entries, treat it as a valid empty spec (all counts = 0).

## Step 3 — Render the output

### Aggregate mode

```
| feature | current phase           | pass | fail | blocked | stale | critical outstanding | last updated |
|---------|-------------------------|------|------|---------|-------|----------------------|--------------|
| <name>  | <N>. <phase> (<status>) | <P>  | <F>  | <B>     | <S>   | <C>                  | <YYYY-MM-DD> |
```

- Sort rows alphabetically by feature name.
- The `current phase` column shows the active phase from the `## Phase progress` table (e.g. `3. review (done)`, `4. fix (in-progress)`, `complete`, `not-started`, or `n/a` for older specs without a Phase progress block). It's the single most useful at-a-glance signal — which feature is waiting on what.
- `pass` / `fail` / `blocked` / `stale` are AC counts from the per-AC table.
- If a spec has zero AC entries, append ` (empty)` to the feature name.
- The `critical outstanding` column shows the count of `Status: pending` + `Status: deferred` Critical findings in the spec's latest review report. `0` is healthy; `n/a` if no review has run yet; `?` if the report file can't be read.
- After the table, print one summary line:
  `<K> specs, <T_total> ACs — <P> pass / <F> fail / <B> blocked / <S> stale. <C_total> critical findings outstanding across all specs.`

### Drill-down mode

```
| AC-ID    | status      | notes |
|----------|-------------|-------|
| AC-1.1   | pass        |       |
| AC-2.1   | fail        | flake on retry |
```

After the AC table, append a Phase progress block (verbatim mirror of the table from spec-status.md if present; one-line `Phases: n/a` otherwise), then a one-line review summary block:

```
Phase progress:
  1. spec     — done (2026-05-11) — approved by user
  2. build    — done (2026-05-11) — 4/4 ACs pass
  3. review   — done (2026-05-11) — 8 findings (0 critical)
  4. fix      — in-progress (2026-05-11) — 5/8 handled
  5. validate — pending
  6. ship     — pending

Latest review: <path or "(none yet)">
Findings: <fixed>/<total> handled, <critical_outstanding> critical outstanding
```

- Sort the AC table rows by AC-ID (numeric on each side of the dot, not lexical — so `AC-2.1` comes before `AC-10.1`).
- After the review-summary block, print one closing summary line:
  `<feature>: <P>/<T> AC passing, last updated <YYYY-MM-DD>.`

## Step 4 — Constraints

- Never print AC entries from outside `docs/specs/<feature>/spec-status.md`.
- Never modify any file.
- Never invoke other SDD skills or commands. Just render and stop.
- Keep the output to: zero-or-one short heading line, the table, the summary line. No commentary, no "next steps", no emoji.
