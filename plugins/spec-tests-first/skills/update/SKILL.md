---
name: update
description: Iteration handler for the SDD cycle. Use when the user invokes /sdd:update <feature> to amend an existing spec mid-cycle. Snapshots spec.md → spec.md.prev, opens the spec for edits, diffs the two versions to identify changed/added/removed AC-IDs, and marks affected ACs as stale/not-started/removed in spec-status.md so the next /sdd:build iterates only the affected ACs via per-AC red-green-refactor. Does NOT regenerate tests directly — that's /sdd:build's job in STF v2.
argument-hint: <feature-name>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion
---

# /sdd:update — Iterate on an Existing Spec

**Announce at start:** Say to the user: "I'm using /sdd:update to amend `$1`'s spec — snapshot to spec.md.prev, edit, diff, mark changed ACs as stale. Tests get rewritten by `/sdd:build`'s RGR loop, scoped to only the stale + new ACs." Then proceed.

You are running the iteration handler for feature **$1**. This skill exists because real SDD is iterative: implementation reveals spec gaps, requirements shift, and we need a back-edge that does NOT nuke passing tests.

In STF v2, `/sdd:update` no longer regenerates tests — that's `/sdd:build`'s job in the per-AC red-green-refactor loop. This skill's role is narrower: snapshot, edit, diff, mark.

## Iron Law

> **Never renumber AC-IDs or VS-IDs. Never modify `pass` ACs. The whole point of `/sdd:update` is that work already done stays done — only the changed/new criteria iterate again.**

## Steps

### 1. Pre-check

`docs/specs/$1/spec.md` must exist. If not, stop:

> No spec at `docs/specs/$1/spec.md`. Run `/sdd:spec $1` first.

### 2. Snapshot the current spec

Copy `docs/specs/$1/spec.md` → `docs/specs/$1/spec.md.prev`. This is the diff baseline.

```bash
cp docs/specs/$1/spec.md docs/specs/$1/spec.md.prev
```

(Or use the Read tool to read the current spec, then Write to `spec.md.prev` — same effect.)

### 3. Open the spec for edits

Present the current spec to the user and ask what they want to change. Use the same 8-section structure as `/sdd:spec` (Goal, Requirements, Acceptance Criteria, User stories, Technical details, Out of scope, Edge cases, Validation steps). Apply edits to `spec.md` in place.

Critical rules carried from `/sdd:spec`:

- **Never reuse an AC-ID or VS-ID** that already exists. New criteria get the next available ID — never renumber.
- **Every error-path AC names a discriminating signal** (specific stderr substring / exception class / status code). Required by `/sdd:build`'s RED step.
- **Multi-service specs:** preserve the `Services:` line and per-service AC subsections (`### Backend (service: backend)`).

### 4. Diff and identify changed Acceptance Criteria

Compare `spec.md` against `spec.md.prev`. Build three lists of `AC-IDs`:

- **Added** — present in new spec, not in old.
- **Removed** — present in old spec, not in new.
- **Modified** — present in both but text changed (semantically — minor wording cleanups don't count; a discriminating-signal change does).

Print all three lists to the user so they can confirm before any status changes:

```
Spec diff for `$1`:
  Added:    AC-3.2, AC-3.3
  Modified: AC-1.2  (discriminating signal changed: "must be positive" → "must be ≥ 0.01")
  Removed:  AC-4.1
```

AskUserQuestion: confirm / cancel / let user describe more changes to add to the spec.

### 5. Mark in spec-status.md

Open `docs/specs/$1/spec-status.md`. For each AC in the lists above:

- **Added** → set status to `not-started` (with a note "added by /sdd:update YYYY-MM-DD").
- **Modified** → set status to `stale` (with a note "modified by /sdd:update YYYY-MM-DD"). The existing AC-row in the status table stays; its `Status` field changes.
- **Removed** → set status to `removed` (with a note "removed by /sdd:update YYYY-MM-DD"). Keep the row for audit; don't delete it.
- **Unchanged ACs** → leave their existing status alone (`pass`, `fail`, `blocked` are all preserved).

Use targeted Edit calls — never rewrite the whole file.

### 6. Checkpoint

Output:

```
Spec amended.
  Added (not-started):  <list>
  Modified (stale):     <list>
  Removed:              <list>
  Preserved (pass):     <count> ACs unchanged, tests stay green

Next: /sdd:build $1
  /sdd:build will iterate ONLY the not-started and stale ACs via per-AC red-green-refactor.
  Passing ACs and their tests are untouched.
```

DO NOT invoke `/sdd:build` directly — the user explicitly triggers the next phase.

## Rules

- **Never regenerate tests in this skill.** Test scaffolding and per-AC test writing are `/sdd:build`'s job under v2's per-AC RGR loop.
- **Never renumber AC-IDs or VS-IDs.** New criteria get fresh numbers.
- **Never modify `pass` ACs' status.** Their tests stay green; their rows stay `pass`.
- **Never delete `removed` ACs' rows from spec-status.md.** Keep them as `removed` for audit; `/sdd:build` ignores them.
- **Always snapshot to `spec.md.prev` first.** The diff is the source of truth for what changed.
- **Always print the three lists for user confirmation** before mutating spec-status.md.
