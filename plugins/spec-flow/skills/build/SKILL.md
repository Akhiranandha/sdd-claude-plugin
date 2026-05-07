---
name: build
description: Phase 2 of the SDD cycle. Use when the user invokes /spec-flow:build <feature> to implement the spec at docs/specs/<feature>/spec.md story-by-story. Performs git pre-checks (offers git init if missing), scaffolds .gitignore for the language detected, implements bottom-up per User Story, writes per-feature artifacts to docs/specs/<feature>/spec-status.md, updates project-wide docs/codebase-map.md, and prompts a recommended-keep-branch rollback path on hard failure.
argument-hint: <feature-name>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion
---

# /spec-flow:build — Phase 2: Implement the Spec

You are running Phase 2 of the SDD cycle for feature **$1**. Inputs: `docs/specs/$1/spec.md`. Outputs: working code + `docs/specs/$1/spec-status.md` + an updated project-wide `docs/codebase-map.md`.

## Pre-checks

1. **Spec exists?** `docs/specs/$1/spec.md` must exist. If not, stop and tell the user to run `/spec-flow:spec $1`.

2. **Git initialized?** Run `git rev-parse --is-inside-work-tree` (silently). If NOT a git repo, ask via AskUserQuestion:

   > This isn't a git repo. SDD's safety rails (stash, branch, rollback) need git. Choose:
   >
   > - **(a, recommended)** Run `git init` and proceed
   > - **(b)** Proceed without git (no stash / branch / rollback)
   > - **(c)** Cancel

   Wait for an explicit choice. On (a): run `git init`, then continue. On (b): warn and proceed without safety rails. On (c): stop.

3. **`.gitignore` scaffolded?** If the project has no `.gitignore`, scaffold a minimal one for the language detected (look at the spec's Section 4 "Technical details" for the stack). Do this even when git isn't in use yet — a future `git init` then inherits the file and won't accidentally track build artifacts.
   - **Python** → `__pycache__/`, `*.pyc`, `*.pyo`, `.venv/`, `venv/`.
   - **Node / TypeScript** → `node_modules/`, `dist/`, `build/`, `.env`, `coverage/`.
   - **Go** → compiled binaries, `vendor/` (if used).
   - **Other** → ask the user what to ignore.

   If the spec's Section 4 names a runtime data file or DB under "Storage" (e.g. `data.json`, `*.db`), include it in `.gitignore` too. This prevents the `/spec-flow:ship` review from catching runtime artifacts as a CAUTION later.

4. **Git stash + branch (if git is in use).**
   - Run `git status --porcelain`. If the working tree is dirty, run `git stash push -u -m "spec-flow:build pre-start stash for $1"` and tell the user you stashed their WIP — do NOT silently bundle unrelated work into the spec branch.
   - Create a new branch: `git checkout -b feature/$1` (or `git checkout` it if it already exists).

## Step 1 — Gather context

Read in this order:

1. **`docs/specs/$1/spec.md`** — read fully. Note all `US-N` IDs from Section 3 and any Done-when checks (especially discriminating signals named in error-path stories).
2. **`docs/specs/$1/spec-status.md`** — if it exists, read which `US-N`s are already `done` vs need work. **Stories not yet `done`** (status `in-progress`, `blocked`, or `not-started`) are the ones to rebuild this run. `done` stories already work — leave them alone unless the spec changed.
3. **`docs/codebase-map.md`** — the project-wide map. If it exists, use it to find existing modules so you don't duplicate.

After reading, you should be able to state in one sentence what the spec needs and where it goes. If you can't, you don't have enough context yet.

## Step 2 — Plan checkpoint (non-trivial specs only)

If the spec has more than ~3 `US-N` IDs or touches more than ~3 files, present a short plan to the user before writing code:

- Files you'll create / change, in dependency order (bottom-up).
- 5–10 lines max.

Wait for the user to confirm or redirect. For trivial specs, skip the checkpoint.

## Step 3 — Mark in-progress

Open or create `docs/specs/$1/spec-status.md`:

- **If it does not exist**, create it with this template:

  ```markdown
  # spec-status: $1

  Last updated: <today>

  ## Status per User Story

  | US-ID | Status      | Notes                        |
  | ----- | ----------- | ---------------------------- |
  | US-1  | in-progress | (auto from /spec-flow:build) |
  | US-2  | in-progress |                              |
  ```

- **If it exists** (e.g. an update cycle, or after a failed validate), keep `done` stories as `done`. For every story not yet `done` (status `in-progress`, `blocked`, or `not-started`), set it to `in-progress` so this run will work on it. If the spec has any new `US-N` IDs that aren't in spec-status.md yet, append them as `in-progress`.

## Step 4 — Implement, story-by-story, bottom-up

Work through each `US-N` whose status is `in-progress` (after Step 3), in numeric order. Skip stories already `done`. For each story:

1. Build dependency-ordered: shared utilities → domain logic → integration glue → entry points. Keep changes scoped to the spec — no drive-by refactoring.
2. Honor every discriminating signal named in the story's Done-when checks. If a check expects `stderr` to contain `"amount must be positive"`, that exact substring must appear in your error output.
3. After implementing, **self-review** the change against the story's acceptance: does the code actually deliver what the story describes? Run any quick local sanity check the Done-when block describes (a single `Bash` command, an import smoke check) — but do **not** run the full Done-when validation here; that's `/spec-flow:validate`'s job.
4. Update `spec-status.md`: set the story to `done` if implemented and self-review passed; leave `in-progress` if you couldn't finish; set `blocked` with a one-line note if you hit a hard external blocker.

## Step 5 — Write end-of-build artifacts

After all stories are processed, write/update both per-feature artifacts:

### `docs/specs/$1/spec-status.md`

Confirm every `US-N` has its final status (`done` / `in-progress` / `blocked` / `not-started`) and update the `Last updated` timestamp. Append a `## Deviations from spec` section if anything in the implementation diverges from what the spec describes.

### `docs/codebase-map.md` (project-wide, append/merge)

This is **one file for the whole project**, not per-feature. Update it in place — never create `docs/specs/$1/codebase-map.md`.

Procedure:

1. **Read** the existing `docs/codebase-map.md`. If it doesn't exist, create it with the template below.
2. **Collect** every source file this build touched (created or modified) plus the spec's runtime data file (named in Section 4).
3. **Merge** rows into the table:
   - If a file path already has a row, update its role description if this spec changed the file's purpose; otherwise leave it.
   - If the file is new, append a row.
   - Preserve any rows from prior specs whose files weren't touched in this build.
4. **Append** any new invariants under `## Key invariants`. Do not delete prior invariants unless they are now wrong — flag those with a one-line note when you remove them.

Fresh-file template:

```markdown
# codebase-map

Project-wide map of source files and their roles. Updated by `/spec-flow:build` after each spec.

| File     | Role            |
| -------- | --------------- |
| `<path>` | <one-line role> |

## Key invariants

- <one-liner per non-obvious invariant>
```

For monorepos / multi-tier projects, you MAY group rows under `## Server`, `## Client`, `## Shared` headers (each with its own table) instead of one flat table — pick one structure and keep it consistent across builds.

## Failure rollback prompt

If implementation hard-fails (multiple stories blocked, or unrecoverable error), use AskUserQuestion to ask:

> Implementation failed. Choose:
>
> - **(a, recommended)** Keep the branch as-is for debugging
> - **(b)** Reset the branch to the pre-implementation commit
> - **(c)** Delete the branch and restore the stash

Execute the user's choice. Never default to a destructive option.

## Checkpoint on success

When all target stories are `done` (or you've intentionally left some `in-progress` for `/spec-flow:validate`), output:

> Implementation complete. Status saved to `docs/specs/$1/spec-status.md`, files mapped in `docs/codebase-map.md`.
> Run `/spec-flow:validate $1` to walk through each story's Done-when checks.

## Guardrails

- **Artifact paths.** `spec-status.md` is per-feature at `docs/specs/$1/spec-status.md`. `codebase-map.md` is project-wide at `docs/codebase-map.md`. NEVER write `docs/specs/$1/codebase-map.md`. NEVER write to `docs/spec-status.md` or any other location.
- **No drive-by refactoring.** Scope changes to the spec.
- **Never `git push`, `--force`, `--no-verify`, or `git reset --hard`** in this flow. Commit happens in `/spec-flow:ship`.
