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

<!-- Slices 7-10 append below this point -->
