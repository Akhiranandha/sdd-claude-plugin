---
name: build
description: Phase 3 of the SDD cycle. Use when the user invokes /sdd:build <feature> to implement the spec at docs/specs/<feature>/spec.md against pre-written tests at tests/<feature>/. Performs git pre-checks (offers git init if missing), scaffolds .gitignore for the language detected, detects the test runner from the spec's Technical details, runs a baseline to separate target from unrelated failures, implements bottom-up, runs a CAPPED test-fix loop (3 attempts max, no silent retries), writes per-feature artifacts to docs/specs/<feature>/spec-status.md and docs/specs/<feature>/codebase-map.md, and prompts a recommended-keep-branch rollback path on hard failure.
argument-hint: <feature-name>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion, Agent
---

# /sdd:build — Phase 3: Implement the Spec

You are running Phase 3 of the SDD cycle for feature **$1**. Inputs: `docs/specs/$1/spec.md` + `tests/$1/`. Outputs: working code + `docs/specs/$1/spec-status.md` + an updated project-wide `docs/codebase-map.md`.

## Pre-checks

1. **Spec exists?** `docs/specs/$1/spec.md` must exist. If not, stop and tell the user to run `/sdd:spec $1`.

2. **Tests exist?** Confirm `tests/$1/` exists and contains test files. If not, stop and tell the user to run `/sdd:tests $1`.

3. **Git initialized?** Run `git rev-parse --is-inside-work-tree` (silently). If NOT a git repo, ask via AskUserQuestion:

   > This isn't a git repo. SDD's safety rails (stash, branch, rollback) need git. Choose:
   > - **(a, recommended)** Run `git init` and proceed
   > - **(b)** Proceed without git (no stash / branch / rollback)
   > - **(c)** Cancel

   Wait for an explicit choice. On (a): run `git init`, then continue. On (b): warn and proceed without safety rails. On (c): stop.

4. **`.gitignore` scaffolded?** If the project has no `.gitignore`, scaffold a minimal one for the language detected (look at the spec's Section 5 "Technical details" for the stack). Do this even when git isn't in use yet — a future `git init` then inherits the file and won't accidentally track build artifacts.

   - **Python** → `__pycache__/`, `*.pyc`, `*.pyo`, `.pytest_cache/`, `.venv/`, `venv/`, plus any runtime data file mentioned in the spec (e.g. `data.json`, `*.db`).
   - **Node / TypeScript** → `node_modules/`, `dist/`, `build/`, `.env`, `coverage/`.
   - **Go** → compiled binaries, `vendor/` (if used).
   - **Other** → ask the user what to ignore.

   Always exclude the spec's named runtime data file (parse it out of "Technical details" / "Storage"). This prevents the `/sdd:ship` review from catching runtime artifacts as a CAUTION later.

5. **Git stash + branch (if git is in use).**
   - Run `git status --porcelain`. If the working tree is dirty, run `git stash push -u -m "sdd:build pre-start stash for $1"` and tell the user you stashed their WIP — do NOT silently bundle unrelated work into the spec branch.
   - Create a new branch: `git checkout -b feature/$1` (or `git checkout` it if it already exists).

## Step 1 — Gather context

Read in this order:

1. **`docs/specs/$1/spec.md`** — read fully. Note all `AC-IDs` from Section 3 and the discriminating signals named in error-path ACs.
2. **`docs/specs/$1/spec-status.md`** — if it exists, read which `AC-IDs` are already `pass` vs `stale`. In a `/sdd:update` cycle, only `stale` ACs need rebuild; `pass` ACs already work.
3. **`docs/codebase-map.md`** — the project-wide map. If it exists, use it to find existing modules so you don't duplicate.
4. **The test files at `tests/$1/`** — these are the contract. Read them. They name the discriminating signals you must produce.

After reading, you should be able to state in one sentence what the spec needs and where it goes. If you can't, you don't have enough context yet.

## Step 2 — Resolve the test command

The project's test command lives in **`CLAUDE.md`** under a `## Test commands` section as a markdown table — one row per named runner, one marked `default: yes`. This is project-wide config; resolve it once and reuse across phases.

### 2a — Read CLAUDE.md

Look for a heading matching `## Test commands` (case-insensitive). If found, parse the table rows:

```markdown
## Test commands

| name | command              | default |
|------|----------------------|---------|
| unit | `python -m pytest`   | yes     |
| e2e  | `playwright test`    | no      |
```

Pick a row:

- If spec.md Section 5 names a runner by name (e.g. "Test framework: e2e"), pick that row.
- Otherwise, pick the row with `default: yes`. If multiple rows are default, pick the first.

If the chosen command does NOT already include a path scope (e.g. plain `python -m pytest`, plain `npx jest`), append `tests/$1/` to scope it to this feature. Do NOT do this for runners that scope themselves via flags (`mvn -Dtest='*$1*' test`, `./gradlew test --tests '*$1*'`) or for project-wide runners that the user has explicitly scoped (`go test ./...`).

### 2b — If CLAUDE.md is missing or has no `## Test commands` section

Auto-detect from project files:

- `pyproject.toml` or `setup.py` → `python -m pytest`
- `package.json` with a `scripts.test` entry → use that script verbatim (`npm test`); else detect a jest / vitest binary in `node_modules/.bin/`
- `Cargo.toml` → `cargo test`
- `go.mod` → `go test ./...`
- `pom.xml` → `mvn test`

Confirm with the user via AskUserQuestion:

> Detected test command: `<cmd>`. Save this to CLAUDE.md so future SDD phases reuse it?
> - **(a, recommended)** Yes, save and proceed
> - **(b)** Override — let me type a different command (then save that)
> - **(c)** Use it once, don't save (skip CLAUDE.md write)

On (a) / (b): write the `## Test commands` section to CLAUDE.md.

- If CLAUDE.md does NOT exist, create it containing **only** the `## Test commands` section. Do not stub other sections.
- If CLAUDE.md exists, append the `## Test commands` section without modifying any existing content.
- If the section already exists but the table is malformed, replace **only that section** — never touch surrounding content.

### 2c — Cache the resolved command

Save the resolved command as a variable for this run. You'll pass it to the `test-runner` subagent in Steps 3 and 7. The CLAUDE.md write happens once; subsequent phases read from CLAUDE.md without re-prompting.

## Step 3 — Baseline run (via test-runner subagent)

Dispatch the `test-runner` subagent via the Agent tool, with `subagent_type: test-runner` and a prompt body that includes:

- `test_command`: the resolved command from Step 2.
- `test_path`: `tests/$1/`.
- `expected_ac_ids`: the full list of AC-IDs from spec.md Section 3.

The subagent returns ONE fenced JSON block. Parse it (extract the first ```json fenced block from the response, ignoring any preamble or trailing prose — Haiku usually complies with the no-preamble rule but parse defensively). Use the result:

- **Target failures** — entries in `failed` / `errored`. Expected at baseline (TDD). Your job in Step 6 is to make them pass.
- **Unrelated failures** — excluded by the subagent (it only reports tests inside `test_path` *and* with AC-IDs; OOS tests are intentionally dropped).
- **`missing_ac_ids`** — AC-IDs in spec.md with no matching test. Stop and tell the user to run `/sdd:tests $1` to fill the gap before proceeding.
- **`error: "runner_failed"` or `"json_parse_failure"`** — surface `stdout_tail` to the user and stop. Do not proceed to implementation.
- **`exit_code != 0` with no AC failures** — this means an OOS test failed. By design, OOS test results don't block the fix loop; they're verified separately at `/sdd:ship` time. Note it in `stdout_tail` for the user's awareness, but do not stop.

If the suite cannot run at all (compile error, missing dep), the subagent will return `runner_failed`; stop and tell the user. Do not fold unrelated breakage into spec work.

## Step 4 — Plan checkpoint (non-trivial specs only)

If the spec has more than ~3 `AC-IDs` or touches more than ~3 files, present a short plan to the user before writing code:

- Files you'll create / change, in dependency order (bottom-up).
- 5–10 lines max.

Wait for the user to confirm or redirect. For trivial specs, skip the checkpoint.

## Step 5 — Mark in-progress

Open or create `docs/specs/$1/spec-status.md`:

- **If it does not exist**, create it with this template:

  ```markdown
  # spec-status: $1

  Last updated: <today>

  ## Status per Acceptance Criterion

  | AC-ID | Status | Notes |
  |---|---|---|
  | AC-1.1 | in-progress | (auto from /sdd:build) |
  | AC-1.2 | in-progress | |
  ```

- **If it exists** (e.g. an update cycle), set every `stale` AC-ID to `in-progress`. Leave `pass` ACs alone.

## Step 6 — Implement, bottom-up

Build dependency-ordered: shared utilities → domain logic → integration glue → entry points. Keep changes scoped to the spec — no drive-by refactoring.

Honor every discriminating signal the tests assert on. If a test expects `stderr` to contain `"amount must be positive"`, that exact substring must appear in your error output.

## Step 7 — Test-fix loop (cap = 3 attempts, via test-runner subagent)

Each attempt: dispatch the `test-runner` subagent with the same inputs as Step 3 (`test_command`, `test_path`, `expected_ac_ids`). Parse the returned JSON.

Evaluate:

- **`failed` and `errored` are empty**: done → Step 8. (`exit_code` may be non-zero if an OOS test failed — that's expected and not a blocker; OOS coverage is verified at `/sdd:ship`, not here.)
- **At least one entry in `failed` or `errored`:** read its `message` / `error` and `file:line`, locate the offending code, fix it, re-dispatch the subagent. **Up to 3 attempts on the same failing cluster.**
- **After 3 attempts on the same cluster, STOP.** Output:
  - Which AC-IDs still fail (use `failed[].ac_id` and `failed[].file:line` from the JSON).
  - A short summary of each of the 3 attempts: what you tried, what changed, what the failure mode was.
  - The line *"Stopping at the 3-attempt cap — handing back to you."*
  - Mark the affected `AC-ID`s as `fail` or `blocked` in `spec-status.md` — values come directly from the subagent's `failed` / `errored` entries.
  - Skip to the **Failure rollback prompt** below.
- **Same failure twice in a row with no progress** (the subagent's `failed[].message` is unchanged between attempts): stop early — your fix isn't fixing it.

You count attempts yourself (cap = 3 is text-driven). The subagent has no memory between calls.

**Never** disable, skip, weaken, or delete a failing test to make the loop pass. **Never** silently retry beyond 3 attempts. **Never** weaken a discriminating assertion. **Never** parse raw test output yourself — always go through the subagent so the contract stays consistent.

## Step 8 — Write end-of-build artifacts

After the loop completes (success OR cap-hit), write/update both per-feature artifacts:

### `docs/specs/$1/spec-status.md`

Update every `AC-ID` to its final status (`pass` / `fail` / `blocked` / `stale`). The values come from the **last** `test-runner` JSON of the run: AC-IDs in `passed` → `pass`, `failed` → `fail`, `errored` → `blocked`, anything in `missing_ac_ids` → `stale`. Append a `## Test-fix loop log` section if the loop ran more than once (one entry per attempt with the JSON's `failed` / `errored` summary). Append a `## Deviations from spec` section if anything in the implementation diverges from what the spec describes.

### `docs/codebase-map.md` (project-wide, append/merge)

This is **one file for the whole project**, not per-feature. Update it in place — never create `docs/specs/$1/codebase-map.md`.

Procedure:

1. **Read** the existing `docs/codebase-map.md`. If it doesn't exist, create it with the template below.
2. **Collect** every source file this build touched (created or modified) plus the spec's runtime data file (named in Section 5).
3. **Merge** rows into the table:
   - If a file path already has a row, update its role description if this spec changed the file's purpose; otherwise leave it.
   - If the file is new, append a row.
   - Preserve any rows from prior specs whose files weren't touched in this build.
4. **Append** any new invariants under `## Key invariants`. Do not delete prior invariants unless they are now wrong — flag those with a one-line note when you remove them.

Fresh-file template:

```markdown
# codebase-map

Project-wide map of source files and their roles. Updated by `/sdd:build` after each spec.

| File | Role |
|------|------|
| `<path>` | <one-line role> |

## Key invariants

- <one-liner per non-obvious invariant>
```

For monorepos / multi-tier projects, you MAY group rows under `## Server`, `## Client`, `## Shared`, `## Tests` headers (each with its own table) instead of one flat table — pick one structure and keep it consistent across builds.

## Failure rollback prompt

If implementation hard-fails (cap hit on multiple clusters, or unrecoverable error), use AskUserQuestion to ask:

> Implementation failed. Choose:
> - **(a, recommended)** Keep the branch as-is for debugging
> - **(b)** Reset the branch to the pre-implementation commit
> - **(c)** Delete the branch and restore the stash

Execute the user's choice. Never default to a destructive option.

## Checkpoint on success

When all target tests pass, output:

> Implementation complete. Tests pass. Status saved to `docs/specs/$1/spec-status.md`, files mapped in `docs/codebase-map.md`.
> Run `/sdd:validate $1` to walk through the spec's validation steps.

## Guardrails

- **Artifact paths.** `spec-status.md` is per-feature at `docs/specs/$1/spec-status.md`. `codebase-map.md` is project-wide at `docs/codebase-map.md`. NEVER write `docs/specs/$1/codebase-map.md` (legacy per-spec path; removed). NEVER write to `docs/spec-status.md` or any other location.
- **Cap = 3.** No silent retries beyond it.
- **No weakening of failing tests.** Surface the issue to the user instead.
- **No drive-by refactoring.** Scope changes to the spec.
- **Never `git push`, `--force`, `--no-verify`, or `git reset --hard`** in this flow. Commit happens in `/sdd:ship`.
