---
name: init
description: Pre-cycle skill for spec-tests-first. Use when the user invokes /sdd:init to set up STF on a repository — auto-detects whether to run Case A (fresh repo, fresh project), Case B (existing codebase, no specs), or Case C (existing repo with flat specs requiring migration). Seeds CLAUDE.md (## Test commands + ## Test layout), scaffolds docs/codebase-map.md from existing source code using per-profile glob patterns, and for Case C migrates flat docs/specs/*.md files into the v2 subdirectory layout (US-N → AC-N.M conversion with TODO discriminating-signal flags for error-path ACs). Idempotent — re-running skips already-migrated artifacts.
argument-hint: <project-name>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion
---

# /sdd:init — Pre-cycle: Bring a Repo into SDD

**Announce at start:** Say to the user: "I'm using /sdd:init to scan `$1` and decide which of three init paths fits (fresh / existing-codebase / migrate-flat-specs), then run only what's needed." Then proceed.

You are running the pre-cycle initializer for project **$1**. Inputs: the working directory state. Outputs: CLAUDE.md config (test commands + test layout), seeded docs/codebase-map.md, and — only in the migration case — restructured docs/specs/<feature>/ subdirectories with per-feature spec-status.md.

## Iron Law

> **Idempotent or nothing. Every action checks if the artifact already exists; if it does, skip with a printed reason. /sdd:init re-runs must be safe — never overwrite spec.md, spec-status.md, or completed CLAUDE.md sections.**

The whole point of init is to bring an existing project into SDD without destroying anything. Overwriting a live spec-status.md (which may contain in-progress phase tracking) is the worst-case failure mode.

## Pre-checks

1. **Working tree clean.** Run `git status --porcelain`. If dirty AND the changes are unrelated to init artifacts (`docs/specs/`, `docs/codebase-map.md`, `CLAUDE.md`), AskUserQuestion: stash WIP (recommended) / proceed anyway / cancel. On stash → `git stash push -u -m "sdd:init pre-start stash"`. On cancel → stop.
2. **Project name provided.** `$1` is the project's friendly name (used in next-step pointer + commit messages). If `$1` is empty, AskUserQuestion to capture one (default: the repo's root directory name).

## Step 1 — Auto-detect the init case

Determine which case applies by scanning the working directory.

### Detection signals

- **Flat specs present** = at least one `docs/specs/*.md` file (NOT inside a subdirectory). Use Glob: `docs/specs/*.md`.
- **Subdirectory specs present** = at least one `docs/specs/*/spec.md`. Use Glob: `docs/specs/*/spec.md`.
- **Source manifests present** = at least one of these files anywhere up to 3 levels deep: `package.json`, `pom.xml`, `build.gradle`, `build.gradle.kts`, `Cargo.toml`, `go.mod`, `pyproject.toml`, `setup.py`, `Gemfile`, `composer.json`, `*.csproj`.
- **CLAUDE.md `## Test commands` present** = boolean, from grep.

### Case decision tree

```
Flat specs present?
├─ YES → Case C (migration)
└─ NO  → Source manifests present?
         ├─ YES → Case B (existing codebase, no specs)
         └─ NO  → Case A (fresh repo, fresh project)
```

### Mixed-state handling

If BOTH flat specs AND subdirectory specs are present (someone migrated by hand for some features), this is **Case C — partial**. Process only the flat files; existing subdirectory specs are preserved untouched.

If subdirectory specs are present AND no flat specs, the project is **already initialized**. Print a summary of detected features and Phase progress (read each `spec-status.md`), then stop without changes.

### Print the plan to the user

After detection, print:

```
/sdd:init for project `<name>`.

Scanning...
  Source manifests detected: <comma-separated list with their subdirectories>
  docs/specs/ contains: <C_flat> flat files, <C_sub> subdirectories
  CLAUDE.md ## Test commands present: <yes / no>

Plan: Case <A | B | C> — <one-line description>

Continue?
  (a, recommended) Yes — run Case <X>
  (b) Cancel
```

AskUserQuestion. On (a): proceed to Step 2 (case-specific path). On (b): stop.

## Case A — Fresh repo, fresh project

**Triggered when:** no source manifests detected AND no `docs/specs/` directory or it's empty.

### A.1 — Confirm intent

AskUserQuestion:

> Detected an empty project — no source code, no existing specs.
> Is this the intended state for `<name>`?
> - (a, recommended) Yes — set up CLAUDE.md + empty codebase-map.md and exit cleanly
> - (b) No — let me cancel and check what's missing

### A.2 — Git pre-checks

Same shape as `/sdd:build`'s pre-checks 2 and 3:

- If not a git repo, offer `git init` (recommended), proceed without git, or cancel.
- If no `.gitignore` at the project root, scaffold a minimal language-agnostic one:

```
# Local scratch
*.log
*.tmp
.env
.env.*

# OS junk
.DS_Store
Thumbs.db
desktop.ini

# IDE
.vscode/
.idea/
*.swp

# Claude Code session artifacts
chat-session/
```

### A.3 — Resolve initial profile (optional)

AskUserQuestion:

> Want to pick a test framework + layout profile now, or leave that for the first `/sdd:spec` to detect?
> - (a) Pick now — I'll save it to CLAUDE.md
> - (b, recommended) Leave for later — /sdd:spec will detect on first invocation

On (a): present the 12 built-in profiles + `custom`. Save to `CLAUDE.md ## Test commands` + `## Test layout` using the single-service or multi-service shape from `/sdd:build`'s Step 2/3.

On (b): skip — `/sdd:spec` will handle this when invoked.

### A.4 — Scaffold empty codebase-map.md

Write `docs/codebase-map.md` (only if it doesn't exist):

```markdown
# codebase-map

Project-wide map of source files and their roles. Updated by `/sdd:build` after each spec.

| File | Role |
|------|------|

## Key invariants

<!-- TODO: add invariants as the project takes shape -->
```

### A.5 — Print next-step pointer

```
/sdd:init complete — project `<name>` ready for SDD.

What's set up:
  - .gitignore scaffolded (if missing)
  - CLAUDE.md ## Test commands + ## Test layout (if you picked A.3 option (a))
  - docs/codebase-map.md (empty, ready for first build)

Next: /sdd:spec <feature> — write your first feature spec
```

Stop. Skip downstream sections.

## Shared step — Profile detection + CLAUDE.md seeding (Cases B and C)

This section is invoked by Case B (Step B.1) and Case C (Step C.2). It writes `CLAUDE.md ## Test commands` + `## Test layout` once per init invocation; subsequent calls within the same run reuse the cached resolution.

### S.1 — Multi-service detection

Glob for stack manifests in **subdirectories** (not the repo root):

- `**/package.json` (excluding root, excluding `node_modules`)
- `**/pom.xml`
- `**/build.gradle*`
- `**/Cargo.toml`
- `**/go.mod`
- `**/pyproject.toml`
- `**/setup.py`
- `**/*.csproj`
- `**/Gemfile`
- `**/composer.json`

If TWO OR MORE distinct subdirectories contain manifests → monorepo signal. AskUserQuestion:

> Detected monorepo with services:
>   - **<name1>** (<stack1>) at `<subdir1>/`
>   - **<name2>** (<stack2>) at `<subdir2>/`
>
> How should /sdd:init configure CLAUDE.md?
> - (a, recommended) Multi-service — per-service test commands + layout profiles
> - (b) Single-service — pick one to configure; ignore the others
> - (c) Cancel

On (a): record the service list, run S.2 + S.3 per service.
On (b): record the chosen service, run S.2 + S.3 once.
On (c): stop the entire init invocation.

If only the repo root has a manifest (or 0/1 in subdirectories), single-service — run S.2 + S.3 once.

### S.2 — Test command resolution per service

For each service (or the single service):

Derive a candidate command from the manifest:

- `pyproject.toml` / `setup.py` → `python -m pytest`
- `package.json` with a `scripts.test` entry → use that script (`npm test`); else detect a jest / vitest binary in `node_modules/.bin/`
- `Cargo.toml` → `cargo test`
- `go.mod` → `go test ./...`
- `pom.xml` → `mvn test`
- `build.gradle` / `build.gradle.kts` → `./gradlew test`
- `*.csproj` → `dotnet test`
- `Gemfile` with `rspec` → `bundle exec rspec`
- `composer.json` with `phpunit` → `vendor/bin/phpunit`

AskUserQuestion (per service if multi-service):

> Service: **<name>** — detected command: `<cmd>`. Save?
> - (a, recommended) Yes — save and proceed
> - (b) Override — type a different command
> - (c) Skip this service

On (a)/(b): write to `CLAUDE.md ## Test commands`. Use the multi-service or single-service format from `/sdd:build`'s Step 2.

### S.3 — Profile resolution per service

Derive a candidate profile from the resolved command (use the table from `/sdd:build` Step 3 — `pytest → python-pytest`, `jest → js-jest`/`react-jest-rtl`, etc.).

If multiple candidates fit (e.g. `jest` could be plain or React), AskUserQuestion. If no candidate matches, ask for `custom` (4 fields: `tests_root`, `files`, `fixtures`, `feature_anchor`).

Save to `CLAUDE.md ## Test layout` using the same single-service / multi-service shape as `## Test commands`.

After both S.2 and S.3 finish for every service, the CLAUDE.md sections are complete. Subsequent steps (codebase-map seeding, spec migration) read these without re-prompting.

## Shared step — Codebase-map seeding

This section is invoked by Case B (Step B.2) and Case C (Step C.4). Reads `references/codebase-map-patterns.md` for each resolved profile, applies the globs, scaffolds `docs/codebase-map.md`.

### M.1 — Pre-check: already exists?

If `docs/codebase-map.md` already exists:

- If it has rows: AskUserQuestion: append newly discovered files (recommended) / skip / abort init.
- If it's empty (template only): proceed; overwrite the empty body.

On append: skip files already present as rows; only add new ones.

### M.2 — Read the patterns reference for the resolved profile(s)

Use the Read tool on `plugins/spec-tests-first/skills/init/references/codebase-map-patterns.md`. Locate the section matching the profile name (e.g. `## profile: spring-boot-junit5`). Parse the three blocks:

- **Include** patterns
- **Exclude** patterns
- **Role templates**

For multi-service: repeat per service profile.

### M.3 — Glob and filter

For each Include pattern:

- Use the Glob tool from the working directory (or from the service's subdirectory for multi-service — e.g. `frontend/src/**/*.tsx`).
- Collect all matches.

For each Exclude pattern:

- Glob the same way, subtract from the Include set.

### M.4 — Assign role strings

For each remaining file path:

- Test each Role template's pattern (in declared order). On first match, use that role string.
- If a `<noun>` placeholder appears, try to derive it from the file name (e.g. `LoginController.java` → `<noun>` = "login"). If derivation isn't obvious, use the placeholder.
- If no template matched, role = `<!-- TODO: describe role -->`.

### M.5 — Build the codebase-map.md table

For **single-service**, write a flat table:

```markdown
# codebase-map

Project-wide map of source files and their roles. Updated by `/sdd:build` after each spec.

## Source files

| File | Role |
|------|------|
| `src/controllers/auth_controller.py` | REST controller for auth |
| `src/services/auth_service.py` | <!-- TODO: describe role --> |

## Key invariants

<!-- TODO: add invariants -->
```

For **multi-service**, group by service:

```markdown
# codebase-map

Project-wide map of source files and their roles. Updated by `/sdd:build` after each spec.

## Server (service: backend)

| File | Role |
|------|------|
| `backend/src/main/java/com/ex/auth/LoginController.java` | REST controller for login |

## Client (service: frontend)

| File | Role |
|------|------|
| `frontend/src/features/login/LoginForm.tsx` | <!-- TODO: describe role --> |

## Key invariants

<!-- TODO: add invariants -->
```

### M.6 — Print the summary

After writing, print:

```
docs/codebase-map.md seeded:
  Total files: <N>
  By service (if multi-service):
    backend: <X> files
    frontend: <Y> files
  TODO role descriptions: <K> (fill in before /sdd:spec / /sdd:build)
```

## Case B — Existing codebase, no specs yet

**Triggered when:** source manifests detected AND no `docs/specs/*.md` flat files AND no `docs/specs/*/spec.md` subdirectories.

### B.1 — Profile detection + CLAUDE.md seeding

Run the **Shared step — Profile detection + CLAUDE.md seeding** above (Steps S.1, S.2, S.3). After this completes, CLAUDE.md has `## Test commands` + `## Test layout` for every service.

### B.2 — Codebase-map seeding

Run the **Shared step — Codebase-map seeding** above (Steps M.1 through M.6). After this completes, `docs/codebase-map.md` is scaffolded.

### B.3 — Print next-step pointer

```
/sdd:init complete — project `<name>` ready for SDD.

What's set up:
  - CLAUDE.md ## Test commands ↓ <command(s)>
  - CLAUDE.md ## Test layout ↓ <profile(s)>
  - docs/codebase-map.md seeded with <N> files
    → fill in <!-- TODO --> role descriptions before /sdd:spec

Existing source is untouched.

Next: /sdd:spec <feature> — write your first feature spec against the existing baseline
```

Stop.

## Case C — Existing repo with flat specs (migration)

**Triggered when:** at least one `docs/specs/*.md` flat file exists.

This is the most complex path. Run in this order: C.1 catalog → C.2 profile (shared) → C.3 restructure → C.4 codebase-map (shared) → C.5 scan + confirm → C.6 final warnings.

### C.1 — Catalog flat files

Use Glob: `docs/specs/*.md`. For each match:

1. **Derive feature name from filename.** Strip leading numeric prefix (`NN-`) and `.md` suffix: `02-auth.md` → `auth`. Files without numeric prefix use the bare stem.
2. **Read content. Classify:**
   - Look for `## User Stories` / `## 3. User Stories` / `## Acceptance Criteria` / `## 3. Acceptance Criteria`. Presence → **feature spec**.
   - Absence → **project-level doc** (e.g. `00-overview.md`).
3. **Skip already-migrated.** If `docs/specs/<feature>/spec.md` already exists, log `Skipping <feature> — already migrated`. Print after the catalog finishes.

Build two lists: `features_to_migrate`, `project_docs_to_move`.

Print the catalog summary:

```
Catalog:
  Feature specs to migrate (<N>): <list>
  Project-level docs to move (<M>): <list>
  Already-migrated (skipped): <list>
```

AskUserQuestion to confirm:

> Proceed with migration?
> - (a, recommended) Yes
> - (b) Cancel — keep flat files as-is

### C.2 — Profile detection + CLAUDE.md seeding

Run the **Shared step — Profile detection + CLAUDE.md seeding** (Steps S.1, S.2, S.3). Same as Case B.

### C.3 — Restructure + US-N → AC-N.M conversion

Initialize a global US-N counter at 1. Process `features_to_migrate` in **filename sort order** (so `01-...`'s stories get the first IDs).

For each feature:

1. **Create `docs/specs/<feature>/`** directory.
2. **Read the flat file content.**
3. **Transform the title.** Replace any `# Spec NN — Title` / `# <stuff>` with `# Spec: <feature>` (matching the v2 template).
4. **Normalize section headings.** Map old → new where possible:
   - `## 1. Goal` / `## Goal` → keep as `## 1. Goal`.
   - `## 2. Requirements` → `## 2. Requirements`.
   - `## 3. User Stories` / `## User Stories` → renumber to `## 4. User stories` (move down — Section 4 in v2 template).
   - Any old `## Tables` / `## Schema` / `## Design` / `## Technical` content → consolidate under `## 5. Technical details`.
   - `## Out of scope` → `## 6. Out of scope`.
   - `## Edge cases` / `## Open questions` → `## 7. Edge cases / open questions`.
   - Old `## Done-when` sub-bullets per story → become `## 8. Validation steps` with VS-N IDs.
5. **Convert US-N → AC-N.M:**
   - In the original `## 3. User Stories` (now `## 4. User stories`) section, find every `- As a <role>, ...` bullet.
   - Assign it a global US-N from the counter; rewrite the bullet as `- **US-N**: As a <role>, ...`. Increment the counter.
   - For EACH story bullet, create a corresponding **happy-path AC** in a new `## 3. Acceptance Criteria` section above it:
     - Feature-local AC number starts at 1, increments per story within the feature (this is the major number N — different from the global US-N counter).
     - Format: `**AC-N.1:** <action paraphrased> succeeds — <observable success state from the story's "so that <benefit>" clause OR a TODO placeholder>`.
   - For each error path hinted in Requirements / Out-of-scope / Edge cases ("validates email", "rejects bad password", "returns 404"), create error ACs:
     - `**AC-N.2:** <invalid input> is rejected. TODO(needs discriminating signal — see /sdd:init warning at end).`
   - Track every TODO created in a global `todos_found` list for the C.6 final warning.
6. **Write `docs/specs/<feature>/spec.md`** with all sections in v2 order.

After this step, all feature specs are restructured but `spec-status.md` files don't exist yet — those come in C.5 (two-signal scan).

### C.3b — Move project-level docs

For each entry in `project_docs_to_move`:

- Move to `docs/<slug>.md` (top-level docs, not under specs/).
- Use `git mv` if the file is tracked (preserves history); otherwise `Write` + delete the source.

Print: `Moved <count> project-level docs to docs/`.

### C.4 — Codebase-map seeding

Run the **Shared step — Codebase-map seeding** (Steps M.1 through M.6). Same as Case B.

### C.5 — Two-signal implementation scan

For each migrated feature, determine proposed Phase 2 status using two concrete signals.

**Signal 1 — Source-file presence.**

Read the feature's `spec.md` Section 5 (Technical details). Extract any file paths or class/module names mentioned (e.g. `LoginController.java`, `auth/login.js`, `UserService`). For each:

- Glob the codebase-map seeded in C.4 for matching paths.
- Count `present` vs `not present`.

If Section 5 doesn't name any files → record `Section 5 silent — files unknown`.

**Signal 2 — Test-file presence.**

Per the resolved test layout profile:

- `feature_anchor: directory` — Glob the `tests_root` substituted with this feature name (e.g. `tests/auth/**`). Count files.
- `feature_anchor: co-located` — Grep across the codebase for test files (`*_test.go`, `*.spec.ts`, `*Test.java`) whose names contain any of this feature's US-N or AC-N.M IDs.

Count test files found.

**Build the proposed-status table:**

```
Detected implementation status for migrated features:

| feature          | source files     | tests   | proposed Phase 2 status |
|------------------|------------------|---------|--------------------------|
| auth             | 3/3 present      | 2 files | done                     |
| user-and-skills  | 5/5 present      | 4 files | done                     |
| billing          | 1/3 present      | 0 files | pending (partial impl)   |
| admin-dashboard  | 0/2 present      | 0 files | pending (not started)    |
| frontend         | 8/8 present      | 0 files | done — tests TBD         |
| reports          | Section 5 silent | 0 files | pending                  |
```

**Threshold rules:**

- `source_files >= 50% present` AND `tests > 0` → **done**
- `source_files >= 50% present` AND `tests == 0` → **done — tests TBD**
- `source_files < 50% present` → **pending (partial impl)**
- Section 5 silent / source files unknown → **pending**

**AskUserQuestion (single batch confirmation):**

> Choose:
> - (a, recommended) Accept all proposed statuses
> - (b) Override per feature — I'll walk you through each
> - (c) Mark all as pending (conservative — you'll /sdd:build each to verify)
> - (d) Cancel migration (rollback all init changes)
> - Optional: (e) Run tests for done-status features now (confirms test pass) — could be slow

On (a): apply proposed statuses; proceed to spec-status.md writing.
On (b): for each feature, AskUserQuestion individually with current proposed status as default.
On (c): override all to `pending`.
On (d): delete all created subdirectory specs + restore flat files (use `git checkout -- docs/specs/` if tracked; else manual file deletion). Stop.
On (e): for each proposed-`done` feature, dispatch the `test-runner` subagent with that feature's tests. If `failed`/`errored` are empty → keep `done`. If non-empty → downgrade to `stale` and note the failure count.

### C.5b — Write per-feature spec-status.md

After the confirmation, for each migrated feature, write `docs/specs/<feature>/spec-status.md`:

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
```

Phase 2 Status comes from C.5's confirmation. Per-AC table:

- If Phase 2 = `done` → all ACs = `done`.
- If Phase 2 = `pending` → all ACs = `not-started`.
- If Phase 2 = `stale` (from (e) option) → all ACs = `stale`.

### C.6 — Final warnings + next-step pointer

Print the consolidated warnings:

```
/sdd:init migration complete for `<project>`.

Migrated features (<N>): <list>
Project-level docs moved: <list>
Codebase-map.md seeded with <K> files (fill in <!-- TODO --> role descriptions)

⚠ <T> error-path ACs need discriminating signals before /sdd:build can RGR-iterate them:
  - docs/specs/auth/spec.md: AC-1.2 — TODO(needs discriminating signal)
  - docs/specs/billing/spec.md: AC-2.3 — TODO(needs discriminating signal)
  ...

Next steps (in order):
  1. Open the migrated spec files and fill in TODO discriminating signals (or remove error-path ACs if not applicable).
  2. Fill in <!-- TODO --> role descriptions in docs/codebase-map.md.
  3. Run /sdd:status to confirm everything looks right.
  4. For features marked Phase 2 = pending or stale: /sdd:build <feature> to implement / re-verify.
  5. For features marked Phase 2 = done: /sdd:review <feature> to bring under SDD review.
```

Finally, AskUserQuestion:

> Migration verified. Delete the original flat spec files at `docs/specs/*.md`?
> - (a) Yes — they're superseded by the subdirectory specs
> - (b, recommended) No — keep them as `*.md.pre-init` for safety; I'll delete them when satisfied

On (a): delete the flat files.
On (b): rename them to `*.md.pre-init` (use `git mv` if tracked, else regular rename).

Done.

## Idempotency rules

`/sdd:init` must be safe to re-run:

| Condition | Behavior |
|---|---|
| `docs/specs/<feature>/spec.md` already exists | Skip; print `Skipping <feature> — already migrated`. |
| `docs/specs/<feature>/spec-status.md` already exists | Skip; preserve all status data (it may contain live tracking from /sdd:build). |
| `docs/codebase-map.md` already exists with rows | AskUserQuestion: append new files / skip / abort. Never overwrite. |
| `CLAUDE.md ## Test commands` already exists | Skip Step S.2's prompt; use existing config. |
| `CLAUDE.md ## Test layout` already exists | Skip Step S.3's prompt; use existing config. |
| US-N IDs already present in flat spec.md | Don't reassign; use as-is. |
| Mixed state (some specs migrated, others flat) | Process only the still-flat ones; skip migrated. |
| Project is already initialized (subdir specs + no flat specs) | Print status summary; stop without changes. |

## Red Flags — STOP and reset

| Thought | Reality |
|---|---|
| "I'll overwrite the existing spec-status.md to add the Phase progress block" | Never. It may contain live `/sdd:build` / `/sdd:fix` tracking. Skip the feature and move on; the user can add the Phase block manually if needed. |
| "The flat spec doesn't have a User Stories section — I'll just write an empty Section 4" | If a flat file has no stories, it's likely a project-level doc, not a feature spec. Move it to `docs/<name>.md`. |
| "I'll guess discriminating signals for error-path ACs" | Never. Use `TODO(needs discriminating signal — see /sdd:init warning at end)` and surface the TODO in C.6's final warnings. The user supplies the signal. |
| "The user said cancel but I've already written half the migration — I'll keep what's done" | On cancel (D in C.5), revert ALL changes from this run. Use `git checkout -- docs/specs/` and delete created subdirectories. Restore the working tree. |
| "I'll auto-delete the flat files after migration without asking" | Never. AskUserQuestion in C.6. Default to renaming to `*.md.pre-init` (safer). |
| "Test files I find don't have AC-IDs — I'll skip the test signal" | If pre-init tests exist but lack AC-IDs (because they predate v2), count them as "test files present" but flag that they'll need renaming when /sdd:build iterates this feature. Don't drop the signal. |

## Rules

- **Always** confirm with the user before mutating anything outside the working directory's metadata (CLAUDE.md, .gitignore, new docs).
- **Never** overwrite existing `spec.md`, `spec-status.md`, or non-empty `codebase-map.md` without explicit append/replace approval.
- **Always** print the auto-detection result before running case-specific logic — user confirms the case before init proceeds.
- **Always** preserve existing US-N IDs in flat specs; reassign only ones that lack them.
- **Always** surface the TODO discriminating-signal list at the end of Case C — it's the most important thing the user must address before `/sdd:build` can iterate the migrated specs.
- **Never** invoke `/sdd:build`, `/sdd:review`, `/sdd:fix`, `/sdd:validate`, or `/sdd:ship` from this skill. Init is preparatory; downstream phases are user-triggered.
- **System binaries:** `git` is required; nothing else (this skill is fully self-contained).
