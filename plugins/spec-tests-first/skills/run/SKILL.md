---
name: run
description: Orchestrator for the full SDD cycle. Use when the user invokes /spec-tests-first:run <feature> to chain spec → build → review → fix → validate → ship end-to-end. Pauses for explicit user confirmation between every phase (y / edit / skip / stop where applicable). Resumable — detects existing artifacts (spec.md, spec-status.md showing all ACs pass, review report, fix-log progress, validation block) and offers to skip already-completed phases.
argument-hint: <feature-name>
allowed-tools: Read, Write, Edit, Glob, Grep, Skill, Bash, AskUserQuestion
---

# /spec-tests-first:run — Full SDD Cycle Orchestrator (6 phases)

**Announce at start:** Say to the user: "I'm using /spec-tests-first:run to chain the 6-phase cycle for `$1` (spec → build → review → fix → validate → ship), pausing for confirmation between every phase." Then proceed.

You are running the full SDD cycle for feature **$1**. This is the orchestrator. You will run each phase, then **pause for explicit user confirmation** before the next. The user can intervene, edit, or stop at any checkpoint.

The v2 cycle is **six phases**: spec → build → review → fix → validate → ship. An optional **Phase 0 (`/spec-tests-first:init`)** runs before spec for un-initialized repos. The dedicated `/spec-tests-first:tests` phase from v1 was folded into `/spec-tests-first:build`'s per-AC red-green-refactor loop.

## Phase 0 — Init detection (optional, run before Phase 1)

Check if the project has been initialized:

- `CLAUDE.md` exists AND has a `## Test commands` section → considered initialized.
- `docs/specs/$1/spec.md` exists (a v2-shaped feature spec) → considered initialized.
- Either condition true → skip Phase 0, proceed to Phase 1 (resumability check).

If **neither** condition is true, AskUserQuestion:

> This project doesn't appear initialized for SDD:
>   - CLAUDE.md ## Test commands: <yes / no>
>   - docs/specs/$1/spec.md: <yes / no>
>
> Run /spec-tests-first:init first to set up the project?
> - (a, recommended) Yes — run /spec-tests-first:init now, then continue with the cycle
> - (b) Skip — /spec-tests-first:spec will detect what's missing and prompt
> - (c) Cancel

On (a): invoke the `init` skill from this plugin: `Skill(skill="init", args="$1")`. After it returns, proceed to Phase 1.
On (b): proceed to Phase 1; the spec skill will handle any missing config.
On (c): stop.

## Iron Law

> **Never auto-advance. Every phase ends with an explicit AskUserQuestion checkpoint. The user can intervene, edit, or stop at every transition. Even on a clean cycle, the human is in the loop.**

The whole reason `/spec-tests-first:run` exists is to give the user a single command for the cycle while preserving every phase's interrupt point. Auto-advancing destroys that contract.

## Resumability check (run before Phase 1)

Detect existing artifacts and offer to skip already-completed phases:

| Artifact | Phase already done | Default action |
|---|---|---|
| `CLAUDE.md ## Test commands` exists AND `docs/codebase-map.md` exists | Init | Skip Phase 0 prompt — project already initialized |
| `docs/specs/$1/spec.md` | Spec | Offer to skip |
| `docs/specs/$1/spec-status.md` shows all ACs `pass` | Build | Offer to skip |
| `./reports/code-review_$1_*.md` exists AND `spec-status.md` `Latest review:` matches | Review | Offer to skip (re-run optional) |
| All Critical findings in latest report = `fixed` or `skipped` | Fix | Offer to skip |
| `spec-status.md` shows `## Validation` block with all VS-N pass | Validate | Offer to skip |

For each detected artifact, AskUserQuestion: *"[Phase X] artifact already exists. Skip and use existing?"* — default yes.

## Cycle

### Phase 1 — Spec

Invoke the `spec` skill: `Skill(skill="spec", args="$1")`. After it completes, AskUserQuestion:

> Spec done. Continue?
> - **(y)** Continue to build
> - **(edit)** I'll revise `docs/specs/$1/spec.md` manually first
> - **(stop)** End the cycle here

On `edit` → tell user to revise the file, then re-run `/spec-tests-first:run $1`. Stop here.
On `stop` → end.
On `y` → continue.

### Phase 2 — Build

Invoke the `build` skill: `Skill(skill="build", args="$1")`. This phase owns:
- Git init / branch / stash safety rails.
- Test framework + layout profile resolution (saves to `CLAUDE.md`).
- Test scaffolding.
- Per-AC red-green-refactor loop.
- Single end-of-build commit.

Let its internal prompts (git init, profile detection, plan checkpoint, cluster-boundary checkpoints, failure rollback) run unchanged.

After it returns, AskUserQuestion:

> Build complete. Continue?
> - **(y)** Continue to review
> - **(stop)** End the cycle here

### Phase 3 — Review

Invoke the `review` skill: `Skill(skill="review", args="$1")`. This dispatches the two reviewer agents in parallel, aggregates via the reporter, writes a timestamped report, and updates `spec-status.md`'s `Latest review:` pointer.

After it returns (and prints the report path + finding counts), AskUserQuestion:

> Review complete. Continue?
> - **(y)** Continue to fix (walk findings one-by-one)
> - **(skip)** Jump straight to validate — proceed without fixing (you can `/spec-tests-first:fix` later)
> - **(stop)** End the cycle here

On `skip` → go directly to Phase 5 (Validate). The Critical gate in `/spec-tests-first:ship` will block if any Criticals are still pending; the user has been warned.

### Phase 4 — Fix

Invoke the `fix` skill: `Skill(skill="fix", args="$1")`. This walks findings severity-then-file, applies edits with revert-on-regression, atomic-commits each fix, and mutates the report in place.

The `fix` skill ends with its own optional re-review prompt (Step 7 of /spec-tests-first:fix). If the user accepts, it dispatches `review` again, which may loop back into `fix` if new Criticals surface. Let the loop run; come back here when it exits.

After it returns, AskUserQuestion:

> Fix complete. Continue?
> - **(y)** Continue to validate
> - **(stop)** End the cycle here

### Phase 5 — Validate

Invoke the `validate` skill: `Skill(skill="validate", args="$1")`. This walks VS-N steps from the spec (automated commands + manual user-confirmed checks). On any VS-N fail, the validate skill appends details to `spec-status.md` and tells the user to investigate.

If validation fails, stop here — let the user fix and re-run. If all pass, AskUserQuestion:

> Validation passed. Continue?
> - **(y)** Continue to ship
> - **(stop)** End the cycle here

### Phase 6 — Ship

Invoke the `ship` skill: `Skill(skill="ship", args="$1")`. This is the inline commit/push/PR/merge phase with SDD-aware metadata. It handles the no-remote case, the commit-message confirmation, the push, the PR creation, the merge style detection, and merge permission gating internally.

## End

When all phases complete (or the user stops at any checkpoint), summarize:

```
SDD cycle complete for `$1`.
  Spec:       docs/specs/$1/spec.md
  Status:     docs/specs/$1/spec-status.md
  Review:     <Latest review path>
  Branch:     feature/$1
  PR / SHA:   <URL or short SHA>
```

If stopped before ship, list what's done and what's pending (the next phase to run).

## Rules

- **Never auto-advance.** Every phase ends with explicit user confirmation.
- **Never skip the Critical gate.** Even if the user picks `skip` on Phase 3 (Review), `/spec-tests-first:ship`'s pre-check still enforces "no pending/deferred Critical findings". If they're outstanding at ship time, ship blocks.
- **Resumability respects the user's intent.** If `/spec-tests-first:run` is re-invoked and finds existing artifacts, offer to skip — never silently re-run a completed phase.
- **Never invoke the deprecated `/spec-tests-first:tests` shim** — test scaffolding is `/spec-tests-first:build`'s job.
