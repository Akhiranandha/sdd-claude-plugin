---

name: run
description: Orchestrator for the full SDD cycle. Use when the user invokes /spec-flow:run <feature> to chain spec → build → validate → ship end-to-end. Pauses for explicit user confirmation between every phase (y / edit / stop). Resumable — detects existing artifacts (spec.md, spec-status.md) and offers to skip already-completed phases.
argument-hint: <feature-name>
allowed-tools: Read, Write, Edit, Glob, Grep, Skill, Bash, AskUserQuestion

---

# /spec-flow:run — Full SDD Cycle Orchestrator

You are running the full SDD cycle for feature **$1**. This is the orchestrator. You will run each phase, then **pause for explicit user confirmation** before the next phase. The user can intervene, edit, or stop at any checkpoint.

## Resumability check (run before Phase 1)

Before starting, detect existing artifacts and ask whether to skip already-completed phases:

| Artifact present                                | Phase already done | Default action |
| ----------------------------------------------- | ------------------ | -------------- |
| `docs/specs/$1/spec.md`                         | Spec               | Offer to skip  |
| `docs/specs/$1/spec-status.md` shows all `done` | Build + Validate   | Offer to skip  |

Use AskUserQuestion to ask: _"[Phase X] artifact already exists. Skip and use existing? (y/n)"_ — default `y`.

## Cycle

### Phase 1 — Spec

Run `/spec-flow:spec $1` (invoke the `spec` skill from this plugin). After it completes, ask:

> Spec done. Continue to implementation? **(y / edit / stop)**
>
> - **edit** → tell user to revise `docs/specs/$1/spec.md` manually, then re-run `/spec-flow:run $1`. Stop here.
> - **stop** → end.
> - **y** → continue to Phase 2.

### Phase 2 — Build

Run `/spec-flow:build $1`. (This phase has its own internal git-init and rollback prompts — let them run.) Ask:

> Build complete. Continue to validation? **(y / stop)**

### Phase 3 — Validate

Run `/spec-flow:validate $1`. If anything fails, stop here and let the user fix (validation already prints next-step guidance). If all pass, ask:

> Validation passed. Continue to ship? **(y / stop)**

### Phase 4 — Ship

Run `/spec-flow:ship $1`. (This phase has its own commit / PR / merge gates — let them run.)

## End

When all phases complete (or the user stops at any checkpoint), summarize what's done and what's left.
