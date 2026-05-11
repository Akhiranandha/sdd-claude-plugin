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

<!-- Slices 3-10 append below this point -->
