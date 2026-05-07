# sdd — Spec-Driven Development for Claude Code

A Claude Code plugin that runs a Spec-Driven Development cycle as a flat set of seven skills (one per phase), plus a read-only `/sdd:status` slash command and a `test-runner` subagent invoked by the test phases. Each phase is self-contained — there's no orchestrator/worker indirection, and the plugin does not depend on any user-level skills. The only external dependencies are two marketplace plugins used by the ship phase.

## The cycle

```
/sdd:spec <feature>     →  Phase 1: Write spec interactively (8-section template)
/sdd:tests <feature>    →  Phase 2: Generate failing tests scoped to AC-IDs (TDD)
/sdd:build <feature>    →  Phase 3: Implement bottom-up, cap=3 on test-fix loop
/sdd:validate <feature> →  Phase 4: Walk validation steps (auto + manual VS-N checklist)
/sdd:ship <feature>     →  Phase 5: SDD-aware pre-commit review → commit/push/PR → /code-review on PR → merge

/sdd:update <feature>   →  Iteration: amend spec, regenerate ONLY changed-AC tests
/sdd:run <feature>      →  Orchestrator: runs all phases with checkpoints between

/sdd:status [<feature>] →  Inspect: aggregate pass/fail rollup, or per-AC drill-down for one feature (read-only)
```

`/sdd:status` is a slash command, not a skill — it never writes files, never invokes other phases, and never proposes next actions. Use it to check progress at any point in the cycle without disturbing it. With no argument it scans every `docs/specs/*/spec-status.md` and prints one row per spec; with a feature name it drills into that spec and prints one row per AC-ID.

## Spec file convention

All specs live at `docs/specs/<feature>/spec.md` in the target project (not the plugin). Each spec folder also accumulates:

- `spec.md` — the spec itself (8 sections: Goal, Requirements, Acceptance Criteria with AC-IDs, User stories, Technical details, Out of scope, Edge cases, Validation steps with VS-IDs)
- `spec.md.prev` — snapshot from the last `/sdd:update` (used for diff)
- `spec-status.md` — live status per AC-ID (`pass` / `fail` / `blocked` / `stale`)

In addition, the project has **one shared** `docs/codebase-map.md` — a project-wide table of source files with one-line role descriptions, append/merge-updated by `/sdd:build` after each spec.

## Project convention — `CLAUDE.md`

The test command is project-wide config and lives in the project's `CLAUDE.md` under a `## Test commands` section, formatted as a markdown table:

```markdown
## Test commands

| name | command              | default |
|------|----------------------|---------|
| unit | `python -m pytest`   | yes     |
| e2e  | `playwright test`    | no      |
```

- One row per named runner. One row marked `default: yes`.
- `/sdd:tests` and `/sdd:build` resolve the command from this table. `/sdd:build` then scopes it to the current feature (e.g. appends `tests/<feature>/` for runners that don't self-scope via flags).
- Spec.md Section 5 ("Technical details") can name a non-default runner by name (e.g. "Test framework: e2e") to override per-feature without editing CLAUDE.md.
- First-time discovery: if the section is missing, the first phase to run (`/sdd:tests` or `/sdd:build`) auto-detects from `pyproject.toml` / `package.json` / `Cargo.toml` / `go.mod` / `pom.xml`, confirms via AskUserQuestion, and writes the section to CLAUDE.md (creating CLAUDE.md if absent). Subsequent phases read without re-prompting.

## Dependencies

### External plugins

Install these once before using `/sdd:ship` (the only phase that talks to the outside world):

```
/plugin install commit-commands@claude-plugins-official
/plugin install code-review@claude-plugins-official
```

- `commit-commands` provides `commit-commands:commit` (used when there's no remote) and `commit-commands:commit-push-pr` (used when there is one).
- `code-review` provides the `/code-review` slash command (run on the PR after push).

`/sdd:spec`, `/sdd:tests`, `/sdd:build`, `/sdd:validate`, `/sdd:update`, and `/sdd:run` have **no external skill dependencies** — they're entirely self-contained.

### System binaries

- `git` — required by `/sdd:build` (stash, branch, rollback) and `/sdd:ship` (commit, remote detection).
- `gh` (GitHub CLI), authenticated — required by `/sdd:ship` only when a remote is configured (push, PR, merge, code-review-on-PR).
- Project-appropriate language runtime + test runner — resolved from the project's `CLAUDE.md` `## Test commands` section (see "Project convention" above), with spec.md Section 5 as an optional per-feature override. The plugin itself is language-agnostic.

## Subagents

- **`test-runner`** (Haiku 4.5, Bash-only) — Runs the resolved test command, parses the output, and returns a strict JSON summary keyed by AC-ID (`passed` / `failed` / `errored` / `missing_ac_ids` / `exit_code` / `stdout_tail`). Invoked by `/sdd:build` for the baseline and each fix-loop attempt, and by `/sdd:tests` for the pre-implementation baseline. Read-only — never writes files; the parent skill owns `spec-status.md`. Users do not invoke it directly.

## Key design decisions

- **One skill per phase.** No orchestrator/worker split. Each phase's skill contains all the logic for that phase. Easier to audit, fewer cross-references, no convention drift between layers.
- **Test-fix loop cap = 3.** On cap-hit, `/sdd:build` reports failures and stops. No silent retries.
- **Test execution is delegated to a Haiku subagent.** `/sdd:build` and `/sdd:tests` dispatch the `test-runner` subagent for every test run. Raw stdout/stderr stays in the subagent's context; the parent skill consumes a strict JSON contract. Keeps the main conversation clean across the 3-attempt loop and gives parsing a versionable schema.
- **Test command persisted in `CLAUDE.md`.** Resolved once on first run and reused — no per-spec re-discovery. Spec.md Section 5 stays as a per-feature override path.
- **Acceptance Criteria are split from Requirements.** Requirements are capability statements; Acceptance Criteria are binary, testable, and carry `AC-N.M` IDs. Tests trace back to AC-IDs.
- **Validation steps carry `VS-N` IDs.** Failure logs reference IDs, so the spec can be reordered without breaking history.
- **Negative-path tests must discriminate.** Tests for "X is rejected" must assert on a specific stderr substring / exception class / status — never exit code alone. `/sdd:tests` runs a baseline pre-implementation check to flag false-pass tests.
- **`.gitignore` scaffolded at build time.** `/sdd:build` adds a language-aware `.gitignore` if missing, including the spec's runtime data file. Catches the artifact-leak problem before commit.
- **Two-stage review on ship.** SDD-aware pre-commit review on the local diff (BLOCK/CAUTION/GO verdict, blocks bad commits) → commit/push/PR → `/code-review` on the PR (multi-agent, deeper). Both gates can refuse to proceed.
- **Iterative back-edge.** `/sdd:update` regenerates tests only for AC-IDs that changed, never wiping passing tests.
- **Multi-feature handling.** Sub-capabilities of one feature go in one spec as separate AC-IDs; only genuinely independent features get separate specs. Specs are implemented sequentially.
- **Git is required for safety rails.** `/sdd:build` offers `git init` if missing; without git there's no stash, branch, or rollback.
- **No-remote handling.** `/sdd:ship` detects missing remote and offers commit-only / configure-then-proceed / cancel.


