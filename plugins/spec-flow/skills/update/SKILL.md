---
name: update
description: Iteration handler for the SDD cycle. Use when the user invokes /spec-flow:update <feature> to amend an existing spec mid-cycle. Snapshots the current spec to spec.md.prev, lets the user revise interactively, diffs the two versions to identify changed US-N IDs, and marks affected stories as in-progress in spec-status.md so the next /spec-flow:build only redoes what changed.
argument-hint: <feature-name>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion
---

# /spec-flow:update — Iterate on an Existing Spec

You are running the iteration handler for feature **$1**. This skill exists because real SDD is iterative: implementation reveals spec gaps, requirements shift, and we need a back-edge that does NOT nuke already-`done` stories.

## Steps

### 1. Pre-check

`docs/specs/$1/spec.md` must exist. If not, stop and tell the user to run `/spec-flow:spec $1`.

### 2. Snapshot the current spec

Copy `docs/specs/$1/spec.md` → `docs/specs/$1/spec.md.prev`. This is the diff baseline.

### 3. Update the spec

Walk the user through the spec section-by-section. Ask which sections they want to change (Goal, Requirements, User Stories, Technical details, Out of scope, Edge cases). Apply edits in place to `docs/specs/$1/spec.md`. When adding new stories, assign new `US-N` IDs by continuing the sequence — never renumber existing IDs.

### 4. Diff and identify changed stories

Compare `spec.md` against `spec.md.prev`. Build three lists of `US-N` IDs:

- **Added** — present in new spec, not in old.
- **Removed** — present in old spec, not in new.
- **Modified** — present in both but the story text or its Done-when block changed.

Print the three lists to the user so they can confirm before status gets touched.

### 5. Mark affected in spec-status

In `docs/specs/$1/spec-status.md`:

- For each **added** `US-N`: append a new row with status `not-started`.
- For each **modified** `US-N`: set status to `in-progress` (the implementation may need rework).
- For each **removed** `US-N`: delete the row (after confirming with the user).

Leave unchanged stories alone — `done` stories that didn't change stay `done`.

### 6. Checkpoint

Output:

> Spec updated. Affected stories: **added [list], modified [list], removed [list]**.
> Run `/spec-flow:build $1` to rebuild only the affected pieces.
