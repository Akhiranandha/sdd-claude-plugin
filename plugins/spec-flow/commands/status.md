---
name: status
description: Show SDD progress across all specs in the current project, or drill into one feature's per-story status. Reads docs/specs/*/spec-status.md and renders a done/in-progress/blocked/not-started summary table. Read-only — never modifies files.
argument-hint: "[<feature>]"
allowed-tools: Read, Glob, Bash
---

# /spec-flow:status — SDD progress summary

You are running `/spec-flow:status` against the user's current project (NOT the plugin directory). Output: a single markdown table plus one summary line. This command is **read-only** — do not write files, do not propose next actions beyond the summary line.

## Argument

`$1` — optional feature name.

- **No arg → aggregate mode**: one row per spec across the whole project.
- **`<feature>` arg → drill-down mode**: one row per US-ID inside that feature's spec.

## Step 1 — Locate spec-status.md files

Use Glob, relative to the user's current working directory:

- Aggregate mode: pattern `docs/specs/*/spec-status.md`
- Drill-down mode: pattern `docs/specs/$1/spec-status.md`

If the glob returns nothing:

- Aggregate mode: print exactly `No specs found under docs/specs/. Run /spec-flow:spec <feature> to create one.` and stop.
- Drill-down mode: print exactly `No spec-status.md for feature "$1". Either the feature doesn't exist or /spec-flow:build hasn't run yet.` and stop.

## Step 2 — Parse each spec-status.md

For every matched file:

1. **Feature name** = the directory name directly under `docs/specs/`.
2. **Story entries**: scan the file line-by-line and match `US-N` (regex `US-\d+`) followed — anywhere in the same line, after any separator characters — by one of the status keywords: `done`, `in-progress`, `blocked`, `not-started`. Match case-insensitively.
   - Be format-tolerant: markdown tables, bullet lists, YAML, or plain prose are all fine. You're matching `(US-ID, status)` pairs, not parsing a specific structure.
   - If the same US-ID appears multiple times, the **last** occurrence wins.
   - Lines that contain a US-ID but no recognized status keyword: skip silently.
3. **Last updated**: the file's modification time. Get it with Bash. On Windows use `powershell -Command "(Get-Item '<path>').LastWriteTime.ToString('yyyy-MM-dd')"`. On POSIX use `stat -c %y '<path>' | cut -d' ' -f1`. If neither works, write `unknown`.
4. **Notes** (drill-down only): the rest of the matched line after the status keyword, trimmed. Strip leading separators (`|`, `-`, `:`, whitespace). Empty string if there's nothing left.

If a file exists but contains zero matched story entries, treat it as a valid empty spec (all counts = 0).

## Step 3 — Render the output

### Aggregate mode

```
| feature | total | done | in-progress | blocked | not-started | last updated |
|---------|-------|------|-------------|---------|-------------|--------------|
| <name>  | <T>   | <D>  | <I>         | <B>     | <N>         | <YYYY-MM-DD> |
```

- Sort rows alphabetically by feature name.
- If a spec has zero entries, render its row with all zeros and append ` (empty)` to the feature name.
- After the table, print one summary line:
  `<K> specs, <T_total> stories — <D> done / <I> in-progress / <B> blocked / <N> not-started.`

### Drill-down mode

```
| US-ID | status      | notes |
|-------|-------------|-------|
| US-1  | done        |       |
| US-2  | blocked     | manual check failed on retry |
```

- Sort rows by US-ID numerically (so `US-2` comes before `US-10`).
- After the table, print one summary line:
  `<feature>: <D>/<T> stories done, last updated <YYYY-MM-DD>.`

## Step 4 — Constraints

- Never print story entries from outside `docs/specs/<feature>/spec-status.md`.
- Never modify any file.
- Never invoke other plugin skills or commands. Just render and stop.
- Keep the output to: zero-or-one short heading line, the table, the summary line. No commentary, no "next steps", no emoji.
