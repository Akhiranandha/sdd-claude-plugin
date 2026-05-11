# /sdd:init — Design Document

**Date:** 2026-05-11
**Plugin:** `spec-tests-first` (target version: 2.2.0)
**Status:** Design — pending user approval before plan doc

## Summary

Add a new `/sdd:init <project-name>` skill that bridges the gap between STF v2's assumed greenfield workflow and the reality of using STF on existing repositories. The skill auto-detects which of **three init cases** applies and runs the appropriate path:

- **Case A — Fresh repo, fresh project.** No source, no specs. Init seeds `CLAUDE.md`'s `## Test commands` + `## Test layout` sections and tells the user to run `/sdd:spec`. Near-no-op.
- **Case B — Existing codebase, no specs.** Real source code exists; the user wants to start using STF for *new* features. Init detects stacks (multi-service for monorepos), resolves test profiles, scaffolds `docs/codebase-map.md` from existing source (with TODO placeholders), and seeds `CLAUDE.md`. No specs are written — those come from `/sdd:spec`.
- **Case C — Existing repo with flat specs.** Pre-existing `docs/specs/*.md` flat files that don't match v2's subdirectory layout (the LKMS case). Init restructures them into `docs/specs/<feature>/spec.md`, converts US-N IDs to AC-N.M format (happy paths deterministic, error paths flagged with `TODO(needs discriminating signal)`), seeds per-feature `spec-status.md` with the `## Phase progress` table, scaffolds `docs/codebase-map.md`, and runs a two-signal scan to propose Phase 2 status (done / pending) per feature with a single user confirmation table.

The skill is **idempotent** — re-running it on a partially-init'd project skips already-migrated features and only acts on what's new.

## Goals

- Make STF v2 usable on existing repositories without requiring a manual migration day.
- Preserve existing test-pass state when migrating from flat specs (don't nuke working tests).
- Detect "already implemented" status from concrete file/test evidence (not keyword guessing).
- Bridge v1 story-driven format (US-N) → v2 AC-driven format (AC-N.M) where possible; flag the gaps where it isn't.
- Stay self-contained — no external plugin dependencies (consistent with v2).

## Non-goals

- Run tests automatically as part of init. Test execution is opt-in (default: just check whether test files exist; don't actually run them — could cost minutes on a large project).
- Generate spec content from source code. Init only restructures existing spec text and seeds skeleton files. The user fills in TODO signals and codebase-map role descriptions afterward.
- Migrate from other SDD-style frameworks (e.g. agile-style story files in JIRA, RST specs). v2.2 scope is flat Markdown files following the loose "title + sections" shape captured in `sdd-init-notes.md`.
- Detect partial-implementation gradients ("AC-1.1 done, AC-1.2 pending within the same feature"). Init treats Phase 2 status as feature-level binary (`done` / `pending`); the per-AC table within `spec-status.md` is populated at `not-started` and gets refined by `/sdd:build` later.

## The three cases — auto-detection

`/sdd:init <project-name>` reads the working directory and decides:

```
┌──────────────────────────────────────────────────────────────────────────┐
│ docs/specs/ exists AND contains *.md files at the root (flat)?           │
├──────────────────────────────────────────────────────────────────────────┤
│   YES  →  Case C (migration)                                             │
│   NO   →  source code exists at root or in subdirs (any manifest)?       │
│             YES  →  Case B (existing codebase, no specs)                 │
│             NO   →  Case A (fresh)                                       │
└──────────────────────────────────────────────────────────────────────────┘
```

Heuristics:
- "Source code exists" = at least one of `package.json`, `pom.xml`, `build.gradle`, `Cargo.toml`, `go.mod`, `pyproject.toml`, `setup.py`, `Gemfile`, `composer.json`, `*.csproj` in the repo root or up to 2 levels deep.
- "Flat specs exist" = `docs/specs/*.md` files (NOT inside subdirectories — those are already v2-shaped).
- If `docs/specs/<feature>/spec.md` already exists (v2 layout) AND no flat files at `docs/specs/*.md`, init considers the project **already initialized**. Print a status summary and stop.

Edge case: **mixed state** (some specs already in subdirectories, some still flat). Init treats this as **Case C** and migrates only the flat files. Existing subdirectory specs are preserved untouched.

The detection prompt at start:

```
/sdd:init for project `<name>`.

Scanning...
  Source manifests detected: <list — e.g. "package.json (frontend/), pom.xml (backend/)">
  docs/specs/ contains: <count> flat files, <count> subdirectories
  Existing CLAUDE.md ## Test commands? <yes/no>

Plan: Case <A | B | C> — <one-line description of what init will do>

Continue? (a, recommended) yes  (b) cancel
```

User confirms before init makes any changes.

## Case A — Fresh repo, fresh project

**Trigger:** no source manifests detected, no specs.

**Steps:**

1. Confirm with user the project is meant to be empty/greenfield (sanity check).
2. Initialize git if not already (`git init` via AskUserQuestion gate — same as `/sdd:build`'s pre-check 2).
3. Create `.gitignore` with the existing language-aware defaults from `/sdd:build`'s pre-check 3, plus a `chat-session/` entry preemptively (Claude Code session artifacts).
4. Ask the user for the project's primary stack (the test layout profile question). Save `CLAUDE.md ## Test commands` + `## Test layout` upfront so `/sdd:spec` doesn't have to ask later.
5. Create empty `docs/codebase-map.md` with the fresh template (project-wide map, ready to fill in as specs are built).
6. Print next-step pointer:

   ```
   /sdd:init complete — project `<name>` ready for SDD.

   What's set up:
     - CLAUDE.md ## Test commands ↓ <command>
     - CLAUDE.md ## Test layout ↓ profile: <profile>
     - docs/codebase-map.md (empty)
     - .gitignore scaffolded

   Next: /sdd:spec <feature> — write your first feature spec
   ```

**No source-file scanning, no migration.** Case A is fast (under a minute).

## Case B — Existing codebase, no specs yet

**Trigger:** source manifests detected, but no `docs/specs/` flat files and no `docs/specs/<feature>/spec.md` subdirectories.

**Steps:**

1. Same git/.gitignore pre-checks as Case A.
2. **Multi-service detection.** Scan for stack manifests in subdirectories (same logic as `/sdd:spec`'s Step 0). If multiple services detected, AskUserQuestion: configure multi-service (recommended) / single-service / cancel.
3. **Profile resolution.** For each service (or the single service), derive a candidate profile from the test command in the manifest (e.g. `package.json scripts.test` → `js-jest`/`js-vitest`/`react-jest-rtl` family). AskUserQuestion to confirm + save to `CLAUDE.md`.
4. **Codebase-map seeding.** This is the bulk of Case B's work:
   - For each service, look up the profile's file-discovery patterns from the references file (see "References" section below).
   - Glob the source tree for matching files.
   - Build a table row per file with the pattern's role-template as a `<!-- TODO: describe role -->` placeholder.
   - Group rows under `## <Service>` headers in monorepos (or one flat table for single-service).
   - Add a `## Key invariants` section with a single placeholder bullet (`<!-- TODO: add invariants -->`).
   - Skip test files, build outputs, generated files (per the patterns).
5. Print a summary table of detected files + a next-step pointer:

   ```
   /sdd:init complete — project `<name>` ready for SDD.

   What's set up:
     - CLAUDE.md ## Test commands ↓ <per service>
     - CLAUDE.md ## Test layout ↓ <per service>
     - docs/codebase-map.md seeded with <N> files (<X> per service for monorepo)
       → fill in <!-- TODO --> role descriptions before /sdd:spec

   Next: /sdd:spec <feature> — write your first feature spec against existing code
   ```

**Existing source is left untouched.** No tests are added, no code is moved. Init only inventories.

## Case C — Existing repo with flat specs (migration)

**Trigger:** `docs/specs/*.md` flat files exist.

This is the most complex case — restructure, convert, scan, propose.

### Step C.1 — Catalog flat files

Read every `*.md` directly under `docs/specs/`. For each:

- Detect filename pattern `NN-slug.md` → strip leading numeric prefix to derive feature name. `02-auth.md` → feature `auth`.
- Files with no numeric prefix → use bare stem (`overview.md` → feature `overview`).
- Read content; classify:
  - **Feature spec** — has a `## User Stories` / `## 3. User Stories` / `## Acceptance Criteria` section.
  - **Project-level doc** — no Story/AC section, just narrative (e.g. `00-overview.md`, `architecture.md`). These move to `docs/<slug>.md`, not `docs/specs/<slug>/`.

### Step C.2 — Profile resolution + multi-service detection

Same as Case B Step 3 — write `CLAUDE.md ## Test commands` + `## Test layout` per service before any spec migration touches the file tree. Monorepo detection is opt-in (the user might not realize their existing project is a monorepo).

### Step C.3 — Migrate each feature spec

In **filename sort order** (so `01-...` is processed before `02-...`, ensuring the global US-N counter is deterministic):

1. **Skip already-migrated.** If `docs/specs/<feature>/spec.md` exists, print `Skipping <feature> — already in v2 layout` and move on. (Idempotency.)
2. **Create `docs/specs/<feature>/`** directory.
3. **Read the flat file content.** Apply transformations:
   - Title becomes `# Spec: <feature>` (drop any leading numeric prefix in the title).
   - Ensure section headings match the v2 8-section template (`## 1. Goal`, `## 2. Requirements`, `## 3. Acceptance Criteria`, `## 4. User stories`, `## 5. Technical details`, `## 6. Out of scope`, `## 7. Edge cases / open questions`, `## 8. Validation steps`). Renames as needed (e.g. old `## 2. Tables` containing implementation details → `## 5. Technical details`).
4. **Convert US-N → AC-N.M (the v2 conversion).** Iterate over the source spec's `User Stories` section bullets in order:
   - Each story (`- As a <role>, I want to <action>, so that <benefit>.`) becomes the basis for one **happy-path AC** under `## 3. Acceptance Criteria`:
     - Spec.md gets an `## 4. User stories` section with the original US-N text preserved (US-N IDs assigned per a global counter for the project, same as the LKMS migration rules).
     - Spec.md gets an `## 3. Acceptance Criteria` section with **one happy-path `AC-N.1`** per US-N (the major number N matches the story's position in this spec, starting at 1 per feature — not global).
     - The AC text is auto-generated from the story: `**AC-N.1:** <action> succeeds — <observable success state>`. The observable success state is a TODO if the story didn't name one.
   - For every "error path" hinted at in Requirements / Out-of-scope / Edge cases / Done-when sub-bullets, create a corresponding `AC-N.2` (or `.3`, etc.) with a discriminating-signal placeholder:
     - `**AC-N.2:** <invalid input> is rejected. TODO(needs discriminating signal — see init warning at end).`
   - Track every TODO created. Print a consolidated warning at the end of init.
5. **Write Section 8 — Validation steps.** If the source spec has a `Done-when:` block per story (LKMS convention), convert each to a `VS-N` step. Otherwise, scaffold an empty `## 8. Validation steps` section with one placeholder VS per AC cluster (`- **VS-N** *(manual)*: TODO — verify AC-N.* by ...`).
6. **Write `docs/specs/<feature>/spec-status.md`:**

   ```markdown
   # spec-status: <feature>

   Last updated: <YYYY-MM-DD>
   Latest review: (none yet)

   ## Phase progress

   | Phase           | Status      | Updated      | Notes                              |
   |-----------------|-------------|--------------|------------------------------------|
   | 1. spec         | done        | <YYYY-MM-DD> | migrated by /sdd:init              |
   | 2. build        | <X>         | <YYYY-MM-DD> | <"existing impl detected" / "—">   |
   | 3. review       | pending     | —            | —                                  |
   | 4. fix          | pending     | —            | —                                  |
   | 5. validate     | pending     | —            | —                                  |
   | 6. ship         | pending     | —            | —                                  |

   ## Status per Acceptance Criterion

   | AC-ID | Status | Notes |
   |---|---|---|
   <one row per AC-N.M, status = "not-started" or "done" per the two-signal scan>
   ```

   Phase 2 Status and per-AC Status come from Step C.5 (two-signal scan).

### Step C.4 — Codebase-map seeding

Same as Case B Step 4 — scan source by profile, seed `docs/codebase-map.md` with file rows and TODO role descriptions.

### Step C.5 — Two-signal implementation scan + confirmation table

For each migrated feature, determine proposed Phase 2 / per-AC status using **two signals**:

**Signal 1 — Source-file presence.** Read each spec's old `## Technical details` / Section 5 content. Extract any file paths or class/module names mentioned (e.g. `LoginController.java`, `auth/login.js`, `UserService`). For each:
- Search the codebase-map (seeded in Step C.4) for matching paths.
- Count `present` vs `not present`.

**Signal 2 — Test-file presence.** Per the resolved test layout profile:
- For `feature_anchor: directory` profiles: glob the profile's `tests_root` for files matching the feature.
- For `feature_anchor: co-located` profiles: grep all test-pattern files (`*_test.go`, `*.spec.ts`, etc.) for the feature's AC-IDs (e.g. `AC_1_1`, `AC-1.1`).
- Count test files found.

**Proposed status table:**

```
Detected implementation status for migrated features:

| feature          | source files | tests   | proposed Phase 2 status |
|------------------|--------------|---------|--------------------------|
| auth             | 3/3 present  | 2 files | done                     |
| user-and-skills  | 5/5 present  | 4 files | done                     |
| billing          | 1/3 present  | 0 files | pending (partial impl)   |
| admin-dashboard  | 0/2 present  | 0 files | pending (not started)    |
| frontend         | 8/8 present  | 0 files | done — tests TBD         |
```

Threshold rules:
- `source_files >= 50%` AND `tests > 0` → **done**
- `source_files >= 50%` AND `tests == 0` → **done — tests TBD** (Phase 2 done because impl exists; user is reminded that no tests guard it)
- `source_files < 50%` OR `unknown` (Section 5 didn't name files) → **pending**

**Single confirmation question:**

```
Choose:
- (a, recommended) Accept all proposed statuses
- (b) Override per feature — I'll walk you through each
- (c) Mark all as pending (conservative — you'll /sdd:build each to verify)
- (d) Cancel migration (rollback all init changes)
- Optional: (e) Run tests for done-status features now (confirms test pass) — could be slow
```

On (a): apply the proposed statuses, write per-feature `spec-status.md`. Continue to Step C.6.
On (b): for each feature, AskUserQuestion individually (status: done / pending / stale).
On (c): set every Phase 2 = `pending` regardless of signals. User will `/sdd:build` each manually.
On (d): delete all subdirectory specs and spec-status files created so far in this run; restore flat files; abort.
On (e): for each feature proposed as `done`, run the resolved test command scoped to that feature's tests. If tests pass → keep `done`. If tests fail → downgrade to `stale` (impl exists but test is broken). Then proceed to (a).

### Step C.6 — Final warnings + next-step pointer

Print the consolidated warnings:

```
/sdd:init migration complete for `<project>`.

Migrated features: <list>
Project-level docs moved to docs/: <list>
Codebase-map.md seeded with <N> files (fill in <!-- TODO --> role descriptions)

⚠ <K> error-path ACs need discriminating signals before /sdd:build can RGR-iterate them:
  - docs/specs/auth/spec.md: AC-1.2 — TODO(needs discriminating signal)
  - docs/specs/billing/spec.md: AC-2.3 — TODO(needs discriminating signal)
  ...

Next steps (in order):
  1. Open the migrated spec files and fill in TODO discriminating signals (or remove the error-path ACs if they don't apply).
  2. Fill in <!-- TODO --> role descriptions in docs/codebase-map.md.
  3. Run /sdd:status to confirm everything looks right.
  4. For features marked Phase 2 = pending or stale: run /sdd:build <feature> to implement / re-verify.
  5. For features marked Phase 2 = done: run /sdd:review <feature> to bring them under SDD review.
```

Also, **delete the original flat files** after the user confirms the migration looks right — OR leave them in place with a `## Migrated by /sdd:init on <date>` note appended. Default: leave them in place. AskUserQuestion at the very end:

> Migration verified. Delete the original flat spec files at `docs/specs/*.md`?
> - (a) Yes — they're superseded by the subdirectory specs
> - (b, recommended) No — keep them as `*.md.pre-init` for safety; I'll delete them when I'm satisfied

## Two-signal implementation detection — full procedure

(Already covered inline in Step C.5. Recap here for completeness.)

| Signal | What it checks | False positive risk | False negative risk |
|---|---|---|---|
| Source-file presence | Files named in spec's Section 5 exist in codebase | Low — names are explicit | Medium — spec might use class names; codebase-map has file paths |
| Test-file presence | Tests under profile's `tests_root` contain feature AC-IDs in names | Very low — AC-IDs are unique tokens | Medium — pre-init tests don't have AC-IDs because they predate v2 |

The signals are **AND-combined for `done`** and **OR-combined for `pending`** — both have to look good for "done", but either alone can flag "pending."

## File structure

```
plugins/spec-tests-first/skills/init/
├── SKILL.md
└── references/
    └── codebase-map-patterns.md
```

**`SKILL.md`** (~280 lines estimated):
- Frontmatter (allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion).
- Announce-at-start line + Iron Law callout.
- Auto-detection logic (which case).
- Case A / B / C steps inline.
- Two-signal scan logic.
- Idempotency rules.
- Red Flags section.
- Rules.

**`references/codebase-map-patterns.md`** (~180 lines estimated):
- One section per stack profile (12 profiles + custom).
- Each section lists: file globs to include, file globs to exclude, default role template per match.

Example reference section:

```markdown
## profile: spring-boot-junit5

### Include
- `src/main/java/**/*Application.java` — Spring Boot entry point
- `src/main/java/**/*Controller.java` — REST controller
- `src/main/java/**/*Service.java` — business logic
- `src/main/java/**/*Repository.java` — data access (JPA)
- `src/main/java/**/entity/*.java` — JPA entity
- `src/main/java/**/*Config.java` — Spring configuration
- `src/main/resources/db/migration/V*.sql` — Flyway migration
- `src/main/resources/application*.yml` — Spring profile config

### Exclude
- `src/test/**` — test sources
- `target/**` — build output
- `*.class` — compiled bytecode

### Role templates (used when role can't be inferred)
- `*Controller.java` → "REST controller for <noun>"
- `*Service.java` → "business logic for <noun>"
- `*Repository.java` → "JPA repository for <entity>"
- (... etc.)
```

## Idempotency rules

`/sdd:init` must be safe to re-run:

| Condition | Behavior |
|---|---|
| `docs/specs/<feature>/spec.md` already exists | Skip; print `Skipping <feature> — already migrated`. |
| `docs/specs/<feature>/spec-status.md` already exists | Skip; preserve all status data. |
| `docs/codebase-map.md` already exists | Offer to append newly discovered files (per the reference patterns), never overwrite. |
| `CLAUDE.md ## Test commands` already exists | Skip Step (profile prompts); use existing config. |
| US-N IDs already present in flat spec.md | Don't reassign; use as-is. |
| Mixed state (some specs migrated, others flat) | Process only the still-flat ones; skip migrated. |

## Backward compatibility

- **v1 plugin users:** If a project was initialized by the v1 `spec-lean` plugin (story-driven), `/sdd:init` recognizes the format and runs Case C. The migration converts US-N stories to AC-N.M acceptance criteria with `TODO(needs discriminating signal)` flags for error-path ACs.
- **v2 plugin users with no init:** Already on `docs/specs/<feature>/` layout. `/sdd:init` detects the v2 state and reports "already initialized" without changes.
- **v2.1 plugin users:** Same — `/sdd:init` only acts on flat files and missing config.

## Decisions log

Decisions made during the design conversation (in order):

1. **Scope — one merged skill with auto-detection.** `/sdd:init` decides A/B/C, not three separate commands.
2. **Conversion — US-N → AC-N.M with TODO for error-path signals.** Happy-path stories convert deterministically; error-path ACs get `TODO(needs discriminating signal)` and a warning at the end of init.
3. **File structure — mostly flat, one references file.** Main SKILL.md holds decision logic + shared operations; `references/codebase-map-patterns.md` holds the per-profile globs and role templates.
4. **Implementation detection — two-signal scan with single confirmation table.** Source-file presence + test-file presence. User confirms in batch with optional opt-in test-run for proposed-done features.
5. **Idempotency — strict.** Re-run is safe; never overwrites existing artifacts; only acts on what's new.

## Implementation notes (for the plan doc)

Suggested slice breakdown:

1. **Slice 1** — Branch + skill scaffold (skills/init/SKILL.md + skills/init/references/ + skills/init/references/codebase-map-patterns.md empty stubs).
2. **Slice 2** — Auto-detection logic (Case A/B/C decision tree, prompt template).
3. **Slice 3** — Case A implementation (greenfield path).
4. **Slice 4** — Profile detection + CLAUDE.md seeding (shared by B and C).
5. **Slice 5** — Codebase-map seeding logic + the per-profile reference patterns (the big content slice — fill in references/codebase-map-patterns.md).
6. **Slice 6** — Case B end-to-end (greenfield with existing source).
7. **Slice 7** — Case C — flat spec catalog + classification.
8. **Slice 8** — Case C — US-N → AC-N.M conversion + spec restructure.
9. **Slice 9** — Case C — spec-status.md scaffolding with Phase progress.
10. **Slice 10** — Two-signal scan + confirmation table.
11. **Slice 11** — Idempotency hardening + warning consolidation.
12. **Slice 12** — `/sdd:run` integration: add a Phase 0 entry that detects and offers to run `/sdd:init` if needed.
13. **Slice 13** — README + plugin.json bump to 2.2.0 + smoke verification.

Each slice ends with an atomic commit. The references file (`codebase-map-patterns.md`) gets built up incrementally — patterns added in Slice 5 for the 12 built-in profiles plus a `custom` placeholder.

## Out of scope for v2.2 (future considerations)

- **Standalone `/sdd:migrate` for non-init scenarios.** If a user adds a flat spec later (after init), they'd run `/sdd:init` again — but the UX would be confusing because the command name says "init." A future `/sdd:migrate` could handle this cleanly without re-running init for already-set-up parts.
- **AI-generated role descriptions for codebase-map.** Currently TODO placeholders. A future skill could analyze each file's content and propose a one-line role, with user review.
- **Specs in non-Markdown formats.** RST, AsciiDoc, JIRA-exported, etc. Not supported in v2.2.
- **Migration from other SDD plugins** (e.g. agile-spec, structure-driven). Not supported.
