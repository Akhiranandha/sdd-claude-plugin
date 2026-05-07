---
name: status
description: Show SDD progress across all specs in the current project, or drill into one feature's per-AC status. Reads docs/specs/*/spec-status.md and renders a pass/fail/blocked/stale summary table. Read-only — never modifies files.
argument-hint: "[<feature>]"
allowed-tools: Read, Glob, Bash
---

# /sdd:status — SDD progress summary

You are running `/sdd:status` against the user's current project (NOT the plugin directory). Output: a single markdown table plus one summary line. This command is **read-only** — do not write files, do not run tests, do not propose next actions beyond the summary line.

## Argument

`$1` — optional feature name.

- **No arg → aggregate mode**: one row per spec across the whole project.
- **`<feature>` arg → drill-down mode**: one row per AC-ID inside that feature's spec.

## Step 1 — Locate spec-status.md files

Use Glob, relative to the user's current working directory:

- Aggregate mode: pattern `docs/specs/*/spec-status.md`
- Drill-down mode: pattern `docs/specs/$1/spec-status.md`

If the glob returns nothing:

- Aggregate mode: print exactly `No specs found under docs/specs/. Run /sdd:spec <feature> to create one.` and stop.
- Drill-down mode: print exactly `No spec-status.md for feature "$1". Either the feature doesn't exist or /sdd:build hasn't run yet.` and stop.

## Step 2 — Parse each spec-status.md

For every matched file:

1. **Feature name** = the directory name directly under `docs/specs/`.
2. **AC entries**: scan the file line-by-line and match `AC-N.M` (regex `AC-\d+\.\d+`) followed — anywhere in the same line, after any separator characters — by one of the status keywords: `pass`, `fail`, `blocked`, `stale`, `not-started`. Match case-insensitively.
   - Be format-tolerant: markdown tables, bullet lists, YAML, or plain prose are all fine. You're matching `(AC-ID, status)` pairs, not parsing a specific structure.
   - If the same AC-ID appears multiple times, the **last** occurrence wins.
   - Lines that contain an AC-ID but no recognized status keyword: skip silently.
3. **Last updated**: the file's modification time. Get it with Bash. On Windows use `powershell -Command "(Get-Item '<path>').LastWriteTime.ToString('yyyy-MM-dd')"`. On POSIX use `stat -c %y '<path>' | cut -d' ' -f1`. If neither works, write `unknown`.
4. **Notes** (drill-down only): the rest of the matched line after the status keyword, trimmed. Strip leading separators (`|`, `-`, `:`, whitespace). Empty string if there's nothing left.

If a file exists but contains zero matched AC entries, treat it as a valid empty spec (all counts = 0).

## Step 3 — Render the output

### Aggregate mode

```
| feature | total | pass | fail | blocked | stale | not-started | last updated |
|---------|-------|------|------|---------|-------|-------------|--------------|
| <name>  | <T>   | <P>  | <F>  | <B>     | <S>   | <N>         | <YYYY-MM-DD> |
```

- Sort rows alphabetically by feature name.
- If a spec has zero entries, render its row with all zeros and append ` (empty)` to the feature name.
- After the table, print one summary line:
  `<K> specs, <T_total> ACs — <P> pass / <F> fail / <B> blocked / <S> stale / <N> not-started.`

### Drill-down mode

```
| AC-ID    | status      | notes |
|----------|-------------|-------|
| AC-1.1   | pass        |       |
| AC-2.1   | fail        | flake on retry |
```

- Sort rows by AC-ID (numeric on each side of the dot, not lexical — so `AC-2.1` comes before `AC-10.1`).
- After the table, print one summary line:
  `<feature>: <P>/<T> AC passing, last updated <YYYY-MM-DD>.`

## Step 4 — Constraints

- Never print AC entries from outside `docs/specs/<feature>/spec-status.md`.
- Never modify any file.
- Never invoke other SDD skills or commands. Just render and stop.
- Keep the output to: zero-or-one short heading line, the table, the summary line. No commentary, no "next steps", no emoji.
