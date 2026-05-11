---
name: review
description: Phase 3 of the SDD cycle. Use when the user invokes /sdd:review <feature> to run a code-quality + security review on the feature's changed files. Dispatches code-quality-reviewer and security-reviewer in parallel against the file list, aggregates via code-reporter, and writes ./reports/code-review_<feature>_<ts>.md with stable finding-IDs (SEC-NNN / QUA-NNN) and a per-finding `Status: pending` line. Read-only — never edits source files. Updates docs/specs/<feature>/spec-status.md only to set the `Latest review:` pointer to the new report path. Does NOT gate progress (no BLOCK/CAUTION/GO); the Critical gate lives in /sdd:fix.
argument-hint: <feature-name>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Agent
---

# /sdd:review — Phase 3: Code-Quality + Security Review

You are running Phase 3 of the SDD cycle for feature **$1**. Inputs: the feature's source code on the `feature/$1` branch + `docs/specs/$1/spec.md` + `docs/specs/$1/spec-status.md`. Output: `./reports/code-review_$1_<YYYY-MM-DD_HHMMSS>.md`.

## Pre-checks

1. **Spec status complete?** Read `docs/specs/$1/spec-status.md`. Every AC-ID should have status `pass`. If anything is `fail` / `blocked` / `stale` / `in-progress`, stop and tell the user:

   > Build is not complete for `$1` — `spec-status.md` shows non-pass ACs. Run `/sdd:build $1` first.

2. **Tests still green?** Resolve the test command + path (Phase 2 wrote `## Test commands` and `## Test layout` to `CLAUDE.md`; read them — see `/sdd:build`'s Step 2 / Step 3 for parsing rules, including multi-service awareness). Dispatch the `test-runner` subagent once against the feature's full suite to confirm. If `failed` or `errored` is non-empty, stop with:

   > Feature tests are not green. Re-run `/sdd:build $1` (or investigate manually). Review aborted.

   Do NOT proceed to the reviewers if tests aren't green — they'd be reviewing broken code.

## Step 1 — Identify scope

Build the file list to review. Sources:

1. **Changed files on this branch vs main:**

   ```bash
   git diff --name-only main...HEAD
   ```

2. **Cross-reference with `docs/codebase-map.md`** — pick out rows touched/added in this build (build's end-of-build artifact step updates this map; entries added during the most recent `/sdd:build` are the ones to include).

3. **Feature test files.** Per the resolved profile in `CLAUDE.md ## Test layout`:
   - `feature_anchor: directory` → all files under `tests_root` (e.g. `tests/$1/**`, `src/test/java/.../$1/**`).
   - `feature_anchor: file_prefix` → files in `tests_root` matching the per-feature prefix.
   - `feature_anchor: co-located` → grep across the source tree for test files whose names contain any of this feature's AC-IDs (`AC-1.1`, `AC-1.2`, ...). The AC-ID embedding rule from `/sdd:build` makes this deterministic.

4. **Exclude:**
   - Generated files: `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Cargo.lock`, `go.sum`, `*.min.js`, `*.min.css`, snapshot files, generated proto/SDK code.
   - `docs/specs/$1/*` — these are spec artifacts, not source.
   - `.gitignore` itself.
   - Any file the user explicitly excludes (none by default; `--exclude <glob>` could be added later, not for v2).

If the resulting list is empty, stop with:

> No files in scope to review for `$1`. (Did `/sdd:build` run on this branch?)

Print the resolved list (or the count and a head of 10 paths if it's long) so the user can sanity-check the scope before reviewers fire.

## Step 2 — Dispatch reviewers in parallel

In a **single message**, dispatch both reviewer agents via the Agent tool. This is mandatory parallelism — the reviewers don't read each other's output, and sequential dispatch would double the wait time.

Each dispatch passes the scope file list and a short note about the feature. Example dispatch shape (your actual call uses the Agent tool, not pseudocode):

```
Agent[code-quality-reviewer]
  subagent_type: code-quality-reviewer
  prompt: |
    Review the following files for code quality (maintainability, complexity,
    naming, dead code, error handling, test coverage gaps). The scope is the
    feature "$1" — files were changed by /sdd:build to implement
    docs/specs/$1/spec.md.

    Files:
    <file list>

    Produce your standard `# Code Quality Review` Markdown report.

Agent[security-reviewer]
  subagent_type: security-reviewer
  prompt: |
    Review the following files for security issues (OWASP Top 10, CWE,
    injection, auth, crypto, secrets, dependency CVEs where applicable). The
    scope is the feature "$1".

    Files:
    <file list>

    Produce your standard `# Security Review` Markdown report.
```

Each reviewer is tools-restricted to `Read, Grep, Glob, Bash` and will optionally run any scanners present (`eslint`, `ruff`, `gitleaks`, `semgrep`, `npm audit`, etc.) in check-only mode. Missing tools are logged and skipped — never an error.

Wait for both to complete. Capture their full Markdown outputs (the `# Code Quality Review` block and the `# Security Review` block) verbatim.

## Step 3 — Aggregate via the reporter

Dispatch `code-reporter` with both outputs as part of the prompt body:

```
Agent[code-reporter]
  subagent_type: code-reporter
  prompt: |
    Aggregate the two reviewer outputs below into a single timestamped report
    at ./reports/code-review_$1_<timestamp>.md. Embed each reviewer block
    verbatim per your standard rules, and prepend stable finding IDs
    (SEC-NNN, QUA-NNN) with `Status: pending` lines per the agent's
    "Finding IDs and Status column" section.

    Scope: feature "$1" — N files reviewed.

    ## code-quality-reviewer output
    <full Markdown block, verbatim>

    ## security-reviewer output
    <full Markdown block, verbatim>
```

The reporter writes the file and returns the absolute path + a 2–3 sentence summary. Capture the path.

If the reporter returns an error (couldn't write `./reports/`, both inputs empty, etc.), surface the error to the user and stop — do not invent a substitute path.

## Step 4 — Update spec-status.md `Latest review:` pointer

The `Latest review:` field lives in the top-of-file metadata block of `docs/specs/$1/spec-status.md`, between the title and the AC table:

```markdown
# spec-status: <feature>

Last updated: <YYYY-MM-DD>
Latest review: <absolute path to most recent report>

## Status per Acceptance Criterion
...
```

Use the Edit tool:

- If the file already has a `Latest review:` line → replace its value with the new report path.
- If the file does NOT have one → insert one immediately after the `Last updated:` line. Do not touch any other line.

Single targeted edit. Do not rewrite the whole file.

## Step 5 — Brief summary + next-step pointer

Output exactly this format to the user (substitute counts from the reporter's summary line):

```
Review complete for `$1`.
  Report: <absolute path>
  Findings: <C> critical, <H> high, <M> medium, <L> low, <I> informational

Next:
  /sdd:fix $1      — walk findings one-by-one with revert-on-regression (recommended if any critical or high)
  /sdd:validate $1 — proceed without fixing (you can run /sdd:fix later)
```

If both `critical` and `high` are zero, omit the recommended-emphasis and present `/sdd:validate` first.

Do NOT print the report contents back. The user opens the file. Do NOT propose any further automatic action — gating is `/sdd:fix`'s job.

## Design invariants

- **Read-only across source.** This skill never edits source files, tests, the spec, or `codebase-map.md`. The single mutating operation is the targeted `Edit` of the `Latest review:` line in `spec-status.md` — which is metadata, not code.
- **No verdict gating.** This skill always reports the report path and the counts. It never tells the user "you can't continue". The Critical gate lives in `/sdd:fix`. The shipping gate (no outstanding Critical) lives in `/sdd:ship`.
- **Fresh report every invocation.** Re-running `/sdd:review $1` produces a new timestamped file under `./reports/`. The old reports stay on disk for history. The `Latest review:` pointer is updated to the newest.
- **Stable finding IDs per report.** `SEC-001`, `SEC-002`, `QUA-001`, ... are sequential within a single report. They do NOT survive across reports — a fresh review = fresh IDs. `/sdd:fix` is invoked against a specific report and reads IDs from there.
- **Parallel reviewer dispatch is mandatory.** Sequential dispatch is a contract violation; the two agents don't share state, and parallel halves the wall time.
- **Test green is a hard pre-check.** Reviewing broken code wastes review tokens and produces noise. If tests aren't green, abort early.

## Edge cases

| Case | Behavior |
|---|---|
| `./reports/` not writable | Reporter handles this — surfaces the error and asks. `/sdd:review` echoes the reporter's reply and stops. |
| Zero findings from both reviewers | Reporter writes "No high-priority items — both reviewers reported a clean bill." `/sdd:review` summary shows all zeros and recommends `/sdd:validate`. |
| One reviewer's output empty or missing | Reporter embeds a `> Note: No <quality/security>-reviewer output was supplied.` and continues. `/sdd:review` echoes the partial report. |
| Multi-service feature | Scope still comes from `git diff --name-only main...HEAD` — cross-service. Reviewers handle multi-language naturally. |
| Files extremely large (>10k lines) | Reviewers handle internally; no special case here. |

## Rules

- Always dispatch the two reviewers in parallel (single message, two Agent calls).
- Always dispatch the reporter after both return; never write the report yourself.
- Always update `Latest review:` in `spec-status.md` after the reporter succeeds.
- Never gate the user (no BLOCK/CAUTION/GO verdict).
- Never print the full report back to chat — only the path + summary.
- Never invent finding IDs, severities, or remediations — the reporter owns those.
