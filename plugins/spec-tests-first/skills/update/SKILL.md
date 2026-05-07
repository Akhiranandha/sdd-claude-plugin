---
name: update
description: Iteration handler for the SDD cycle. Use when the user invokes /sdd:update <feature> to amend an existing spec mid-cycle. Snapshots the current spec to spec.md.prev, invokes spec-writer in update mode, diffs the two versions to identify changed AC-IDs, regenerates tests for ONLY the affected ACs (preserving passing tests), and marks affected ACs as stale in spec-status.md so the next /sdd:build only redoes what changed.
argument-hint: <feature-name>
allowed-tools: Read, Write, Edit, Glob, Grep, Skill, Bash
---

# /sdd:update — Iterate on an Existing Spec

You are running the iteration handler for feature **$1**. This skill exists because real SDD is iterative: implementation reveals spec gaps, requirements shift, and we need a back-edge that does NOT nuke passing tests.

## Steps

### 1. Pre-check

`docs/specs/$1/spec.md` must exist. If not, stop and tell the user to run `/sdd:spec $1`.

### 2. Snapshot the current spec

Copy `docs/specs/$1/spec.md` → `docs/specs/$1/spec.md.prev`. This is the diff baseline.

### 3. Update the spec

Invoke the **`spec-writer`** skill (bundled inside this plugin at `skills/spec-writer/`) in update mode. Its description explicitly supports amendments ("update the spec for FR-5", "revise the requirements in FR-3"). Let the user revise sections interactively.

### 4. Diff and identify changed Acceptance Criteria

Compare `spec.md` against `spec.md.prev`. Build three lists of `AC-IDs`:

- **Added** — present in new spec, not in old.
- **Removed** — present in old spec, not in new.
- **Modified** — present in both but text changed.

Print the three lists to the user so they can confirm before tests get regenerated.

### 5. Re-scope tests

Invoke the **`spec-test-writer`** skill (bundled at `skills/spec-test-writer/`) **scoped to ONLY the added and modified `AC-IDs`**. Do NOT regenerate tests for unchanged criteria — that destroys tests that already pass. For removed `AC-IDs`, delete the corresponding tests (after confirming with the user).

### 6. Mark stale in spec-status

In `docs/specs/$1/spec-status.md`, mark every affected `AC-ID` as `stale — needs rebuild`. This is the signal `/sdd:build` reads to know what to redo.

### 7. Checkpoint

Output:

> Spec updated. Affected ACs: **added [list], modified [list], removed [list]**. Tests regenerated for added + modified.
> Run `/sdd:build $1` to rebuild only the affected pieces.
