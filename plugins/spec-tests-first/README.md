# spec-tests-first — Spec-Driven Development for Claude Code (v2)

A Claude Code plugin that runs a Spec-Driven Development cycle in **six phases** (spec → build → review → fix → validate → ship), each a self-contained skill, plus `/sdd:init` (pre-cycle setup for existing repos), `/sdd:update` (iteration handler), `/sdd:run` (orchestrator), a read-only `/sdd:status` slash command, and a `/sdd:tests` deprecation shim. Four subagents power the cycle: `test-runner` (Haiku) for test execution and three reviewers (`code-quality-reviewer`, `security-reviewer`, `code-reporter`, all Sonnet) for the in-cycle code-review phase. The plugin is **fully self-contained** — zero external plugin dependencies.

## The cycle

```
/sdd:init     <project>  →  Phase 0 (optional): Bring a repo into SDD — fresh / existing-codebase / migrate-flat-specs
/sdd:spec     <feature>  →  Phase 1: Write spec interactively (8-section template, monorepo-aware, scope check, user-approval gate)
/sdd:build    <feature>  →  Phase 2: Per-AC red-green-refactor — resolve profile, scaffold, iterate ACs
/sdd:review   <feature>  →  Phase 3: code-quality + security reviewers in parallel → timestamped report
/sdd:fix      <feature>  →  Phase 4: Walk findings one-by-one, revert-on-regression, mutate report in place
/sdd:validate <feature>  →  Phase 5: Walk VS-N steps (auto + manual checklist)
/sdd:ship     <feature>  →  Phase 6: Inline commit/push/PR/merge — SDD-aware messages

/sdd:update   <feature>  →  Iteration: amend spec; only changed ACs re-iterate via RGR in /sdd:build
/sdd:run      <feature>  →  Orchestrator: chains the 6 phases (offers /sdd:init at start if needed)

/sdd:status [<feature>]  →  Read-only progress + findings rollup (aggregate or per-AC drill-down)
```

`/sdd:status` is a slash command, not a skill — it never writes files, never invokes other phases, never proposes next actions. Use it to check progress at any point without disturbing the cycle. With no argument it scans every `docs/specs/*/spec-status.md` and prints one row per spec; with a feature name it drills into that spec and prints one row per AC-ID plus a Latest-review summary block.

## What changed in v2.2 (over v2.1)

- **New `/sdd:init` pre-cycle skill.** STF can now be adopted on existing repos, not just greenfield. Auto-detects three cases:
  - **Case A** — fresh repo, fresh project. Near-no-op: scaffolds CLAUDE.md sections + empty codebase-map.md.
  - **Case B** — existing codebase, no specs yet. Resolves test profile(s) per service for monorepos, seeds `docs/codebase-map.md` from source using per-profile glob patterns (12 built-in profiles + custom).
  - **Case C** — existing repo with flat `docs/specs/*.md` files. Migrates into v2 subdirectory layout. Converts US-N IDs → AC-N.M (happy-path stories deterministic; error-path ACs flagged with `TODO(needs discriminating signal)` placeholders). Scaffolds per-feature `spec-status.md` with the v2.1 `## Phase progress` table. Runs a two-signal implementation scan (source files + test files) and proposes Phase 2 status per feature in a single batch confirmation.
- **`/sdd:run` integration.** The orchestrator detects un-initialized projects and offers to run `/sdd:init` before Phase 1.
- **Strict idempotency.** `/sdd:init` is safe to re-run; never overwrites existing specs, spec-status, or non-empty codebase-map files.

## What changed in v2.1 (over v2)

- **Per-phase status tracking.** `spec-status.md` now has a `## Phase progress` table at the top — every phase skill updates its own row (`pending` → `in-progress` → `done` / `fail` / `blocked`). `/sdd:status` aggregate view shows current phase per spec at a glance.
- **Explicit user-approval gate after `/sdd:spec`.** The spec phase ends with an AskUserQuestion — Approve / Edit / Cancel. Only on Approve does Phase 1 = `done`, and `/sdd:build` refuses to run against an unapproved spec.
- **Scope check in `/sdd:spec` (Step 1.5).** Detects oversized requests (multiple independent features mashed into one) and prompts to split into separate sequential specs vs. combine. Prevention beats post-build refactor.
- **Superpowers-style flow on every skill.** Announce-at-start lines, Iron Law callouts near the top, Red Flags + Common Rationalizations tables for the high-temptation skills (`/sdd:build`, `/sdd:fix`).

## What changed in v2 (over v1)

| v1 | v2 |
|---|---|
| 5 phases: spec → tests → build → validate → ship | 6 phases: spec → build → review → fix → validate → ship |
| `/sdd:tests` generated ALL tests upfront, then `/sdd:build` implemented | `/sdd:tests` is a deprecation shim; `/sdd:build` writes tests per-AC inside the red-green-refactor loop |
| Test-fix loop capped at 3 attempts across the whole feature | Cap = 3 attempts **per AC** (faster failure isolation) |
| Inline pre-commit review hand-rolled inside `/sdd:ship` | Dedicated `/sdd:review` phase using the two reviewer agents |
| PR-level review via external `code-review` plugin | Same reviewers run pre-commit in `/sdd:review`; no external plugin needed |
| Commit/PR via external `commit-commands` plugin | Inlined in `/sdd:ship` with SDD-aware commit message + PR body |
| Python-flavored `tests/<feature>/` assumed | 12 built-in stack profiles + `custom` for unknown stacks; multi-service-aware |
| External dependencies: `commit-commands` + `code-review` | **None** — fully self-contained |

## Spec file convention

All specs live at `docs/specs/<feature>/spec.md` in the **target project** (not the plugin). Each spec folder accumulates:

- `spec.md` — the spec itself (8 sections: Goal, Requirements, Acceptance Criteria with AC-IDs, User stories, Technical details, Out of scope, Edge cases / open questions, Validation steps with VS-IDs).
- `spec.md.prev` — snapshot from the last `/sdd:update` (used for diff).
- `spec-status.md` — live status per AC-ID (`pass` / `fail` / `blocked` / `stale` / `not-started` / `removed` / `in-progress`), plus a top-of-file `Latest review: <path>` pointer.

The project has **one shared** `docs/codebase-map.md` — a project-wide table of source files with one-line role descriptions, append/merge-updated by `/sdd:build` after each spec.

Review reports are written to **`./reports/code-review_<feature>_<YYYY-MM-DD_HHMMSS>.md`** in the target project; `/sdd:fix` mutates the per-finding `Status:` field in place and appends a `## Fix log` table.

## Test commands + test layout (project convention — `CLAUDE.md`)

Two project-wide config sections in the target project's `CLAUDE.md`:

```markdown
## Test commands

| name | command              | default |
|------|----------------------|---------|
| unit | `python -m pytest`   | yes     |
| e2e  | `playwright test`    | no      |

## Test layout

profile: python-pytest
tests_root: tests/<feature>/
files: test_<cluster>.py
fixtures: tests/<feature>/conftest.py
feature_anchor: directory
```

`/sdd:build` reads both. If either section is missing, it auto-detects from project manifests (`pyproject.toml`, `package.json`, `pom.xml`, etc.), confirms with the user via AskUserQuestion, and writes the section. Subsequent phases read without re-prompting.

### Built-in test layout profiles

| Profile | tests_root | files | feature_anchor |
|---|---|---|---|
| `python-pytest` | `tests/<feature>/` | `test_<cluster>.py` | directory |
| `python-unittest` | `tests/<feature>/` | `test_<cluster>.py` | directory |
| `js-jest` | `tests/<feature>/` | `<cluster>.test.ts` | directory |
| `js-vitest` | `tests/<feature>/` | `<cluster>.test.ts` | directory |
| `react-jest-rtl` | `src/features/<feature>/__tests__/` | `<Component>.test.tsx` | directory |
| `angular-jasmine` | co-located with source | `<thing>.spec.ts` | co-located |
| `spring-boot-junit5` | `src/test/java/<pkg>/<feature>/` | `<Cluster>Test.java` | directory |
| `go` | same dir as source | `<source>_test.go` | co-located |
| `rust` | `tests/<feature>/` (integration); `#[cfg(test)]` for unit | `<cluster>.rs` | directory |
| `dotnet-xunit` | `<Project>.Tests/<Feature>/` | `<Cluster>Tests.cs` | directory |
| `ruby-rspec` | `spec/<feature>/` | `<cluster>_spec.rb` | directory |
| `phpunit` | `tests/<feature>/` | `<Cluster>Test.php` | directory |
| `custom` | user-supplied | user-supplied | user-supplied |

For unknown stacks, `/sdd:build` prompts the user inline for the four fields (`tests_root`, `files`, `fixtures`, `feature_anchor`) and saves as `profile: custom`. No plugin update needed.

## Monorepo support

When `/sdd:spec` detects multiple stack manifests in subdirectories (e.g. `frontend/package.json` + `backend/pom.xml`), it asks whether to configure multi-service. On yes:

```markdown
## Test commands

### service: frontend
| name | command    | default |
|------|------------|---------|
| unit | `npm test` | yes     |

### service: backend
| name | command    | default |
|------|------------|---------|
| unit | `mvn test` | yes     |

## Test layout

### service: frontend
profile: react-jest-rtl
tests_root: frontend/src/features/<feature>/__tests__/
...

### service: backend
profile: spring-boot-junit5
tests_root: backend/src/test/java/com/example/<feature>/
...
```

Specs add a `Services:` line and per-service AC subsections:

```markdown
# Spec: login-flow

Services: backend, frontend

## 3. Acceptance Criteria

### Backend (service: backend)
- AC-1.1: POST /api/login returns 200 with JWT on valid credentials.
- AC-1.2: POST /api/login returns 401 with `{"error":"invalid credentials"}` on bad password.

### Frontend (service: frontend)
- AC-2.1: LoginForm submits credentials and stores the returned JWT in localStorage.
- AC-2.2: On 401 response, LoginForm shows the inline error "Invalid credentials".
```

`/sdd:build` reads each AC's enclosing subsection header, looks up that service's profile + test command, and writes the test in the right `tests_root`. Cross-service ACs run sequentially.

Single-service repos omit the per-service subheadings — the format degrades to today's single-block layout. No breakage.

## Dependencies

### External plugins

**None.** STF v2 is fully self-contained.

### System binaries

- **`git`** — required by `/sdd:build` (stash, branch, end-of-build commit), `/sdd:fix` (per-fix atomic commits, revert-on-regression), and `/sdd:ship` (commit, push, remote detection).
- **`gh`** (GitHub CLI), authenticated — required by `/sdd:ship` only when a remote is configured (push, PR, merge).
- Project test runners — resolved from `CLAUDE.md ## Test commands` (single-service or multi-service). The plugin itself is language-agnostic.

## Subagents

- **`test-runner`** (Haiku 4.5, Bash-only) — Runs the resolved test command, parses output, returns a strict JSON summary keyed by AC-ID (`passed` / `failed` / `errored` / `missing_ac_ids` / `exit_code` / `stdout_tail`). Invoked by `/sdd:build`'s RED, GREEN, REFACTOR, and REGRESSION-CHECK steps per AC, by `/sdd:review`'s pre-flight test check, and by `/sdd:fix`'s per-fix regression check. Read-only — never writes files; the parent skill owns `spec-status.md`. Users do not invoke it directly.
- **`code-quality-reviewer`** (Sonnet, Read+Grep+Glob+Bash) — SonarQube-style review across 10 dimensions (cyclomatic complexity, function/file length, duplication, naming, dead code, magic numbers, deep nesting, error handling, inconsistent patterns, test coverage gaps). Language-aware idiom checks. Runs `eslint` / `tsc` / `ruff` / `golangci-lint` / etc. in check-only mode if present. Read-only.
- **`security-reviewer`** (Sonnet, Read+Grep+Glob+Bash) — OWASP Top 10 (2021) review (injection, XSS, CSRF, SSRF, XXE, path traversal, auth/authz, crypto, secrets, configuration, input handling). CWE + OWASP refs per finding. Runs `gitleaks`, `semgrep`, `bandit`, `npm audit`, `govulncheck`, `trivy`, etc. when present. Read-only.
- **`code-reporter`** (Sonnet, Read+Write+Bash) — Aggregates the two reviewer outputs into one timestamped Markdown report under `./reports/`. Performs **no analysis of its own** — faithful aggregation only. Emits stable finding IDs (`SEC-NNN`, `QUA-NNN`) and initializes `Status: pending` per finding so `/sdd:fix` can mutate them in place. Only writes the report file; no other writes.

## Key design decisions

- **Per-AC red-green-refactor.** Tests are written one at a time inside `/sdd:build`. Every test is verified failing via `test-runner` before any production code is written (the Iron Law: no production code without a failing test first). Every fix is verified green before moving on. Then the full feature suite runs as a regression check — a passing AC test that breaks a prior AC reverts the implementation.
- **Cap = 3 attempts per AC, not feature-wide.** Each AC fails fast and gets individual attention. No silent retries past cap.
- **Two-stage code review per feature.** `/sdd:review` runs the two reviewer agents in parallel, writes a report. `/sdd:fix` walks findings one-by-one with revert-on-regression and atomic commit per fix. The Critical gate (no `pending` or `deferred` Critical findings) is enforced at both `/sdd:fix` exit and `/sdd:ship` pre-check.
- **Stable finding IDs.** The reporter emits `SEC-001`, `QUA-014`, etc. so `/sdd:fix` can address findings by ID across multiple sessions. `Status:` lines are parser-stable and mutated in place; the original report body stays verbatim.
- **Atomic commits per successful fix.** Clean git history, cheap revert, easy audit. `/sdd:build` commits the whole feature once at the end; `/sdd:fix` adds incremental commits on top; `/sdd:ship` typically has nothing left to commit (only edits made after the last fix).
- **Stack-aware test layout via profiles.** 12 built-in profiles cover ~95% of real-world stacks. Unknown stacks fall back to a `custom` profile saved to `CLAUDE.md`. Co-located profiles (Angular, Go) skip directory scaffolding; AC-ID embedding in test names makes `/sdd:update`'s "find tests for changed ACs" work for them via grep.
- **Multi-service awareness.** Specs can target multiple services in a monorepo. Each AC carries a service tag (via subsection header). `/sdd:build` resolves per-AC service → profile + test command. `/sdd:review` and `/sdd:fix` see the cross-service diff naturally.
- **Spec.md as the contract.** Section 3 ACs are binary and testable; every error-path AC names a discriminating signal (stderr substring, exception class, status code). Vague error contracts produce false-pass tests; the spec phase enforces this.
- **`/sdd:run` is the orchestrator with explicit checkpoints between every phase.** Resumable — re-invoking detects existing artifacts (spec, status, review report, fix log, validation block) and offers to skip already-done phases.
- **`/sdd:status` is read-only.** Never modifies files; never invokes other phases. Aggregate view gains a `Critical Outstanding` column in v2 so blocked specs surface at a glance.
- **Backward compatible with v1 artifacts.** Existing `spec.md` and `spec-status.md` files keep working. The `Latest review:` field is added the first time `/sdd:review` runs. The `/sdd:tests` command stays valid as a deprecation shim that redirects users to `/sdd:build`.
