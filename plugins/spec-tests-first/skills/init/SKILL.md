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

<!-- Slices 2-10 append below this point -->
