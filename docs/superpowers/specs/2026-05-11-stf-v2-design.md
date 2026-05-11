# STF v2 — Design Document

**Date:** 2026-05-11
**Plugin:** `spec-tests-first` (Spec-Driven Development for Claude Code)
**Status:** Design — pending user approval before implementation planning

## Summary

A v2 redesign of the `spec-tests-first` plugin that brings three coordinated improvements:

1. **Rigorous Test-Driven Development.** Replace the "batch tests-first" model with a **per-AC red-green-refactor** loop inside `/spec-tests-first:build`. Every test is watched failing before any production code is written; every fix is verified green before moving on.
2. **Code-review phase in the cycle.** A new `/spec-tests-first:review` phase dispatches the two reviewer agents from the `review-code` plugin (quality + security) in parallel against the feature's changed files and writes a timestamped report to disk.
3. **Interactive fix phase.** A new `/spec-tests-first:fix` phase walks findings one-by-one with revert-on-regression, mutates the report in-place with per-finding status, and gates progress on Critical findings.

The cycle goes from 5 phases (`spec → tests → build → validate → ship`) to **6 phases** (`spec → build → review → fix → validate → ship`); `/spec-tests-first:tests` is folded into `/spec-tests-first:build` because per-AC RGR means tests are written **during** implementation, not before it.

The plugin becomes **fully self-contained** — zero external plugin dependencies. The PR-level `/code-review` is dropped (redundant with `/spec-tests-first:review`) and `commit-commands` is inlined (so commit messages and PR bodies can be SDD-aware: AC-IDs, validation summary, review status).

## Goals

- Make TDD actually rigorous, with mandatory RED verification per behavior.
- Catch quality and security issues **before** any commit, not after.
- Track fix progress in a single audit-trail artifact (the review report).
- Support real-world repository layouts — including monorepos with mixed stacks (e.g., React + Spring Boot).
- Remove external plugin dependencies. One install, full cycle.

## Non-goals

- Re-doing `/spec-tests-first:spec` or `/spec-tests-first:validate`. Both work fine today and only get minor metadata-tracking tweaks.
- Replacing the `test-runner` subagent. The Haiku-based test-output parser stays as is; the per-AC RGR loop just dispatches it more often.
- Building a new code-review system. We reuse the agents from `review-code` (copied into this plugin, not depended on at install time).

## The new cycle

```
/spec-tests-first:spec  → /spec-tests-first:build → /spec-tests-first:review → /spec-tests-first:fix → /spec-tests-first:validate → /spec-tests-first:ship
                (RGR)       (read)       (edit)       (acceptance)    (commit/PR)
```

| Phase | Role | Status in v2 |
|---|---|---|
| `/spec-tests-first:spec` | Interactive 8-section spec writing with AC-IDs and VS-IDs | unchanged |
| `/spec-tests-first:build` | Per-AC red-green-refactor: write 1 test → watch RED → minimal impl → watch GREEN → refactor → next AC. Owns framework detection, test scaffolding, `spec-status.md`, `codebase-map.md`. | **major rewrite** |
| `/spec-tests-first:review` | Dispatch `code-quality-reviewer` + `security-reviewer` in parallel → `code-reporter` writes `./reports/code-review_<feature>_<ts>.md`. Read-only. No verdict gating. | **new** |
| `/spec-tests-first:fix` | Read latest report → walk findings severity-then-file → per finding: ask `fix / custom / skip / defer` → apply edit → re-run AC tests → revert if any AC regresses → mark status in report → atomic commit per successful fix → optional re-run `/spec-tests-first:review` at the end (recommended). | **new** |
| `/spec-tests-first:validate` | Walk VS-N steps (auto + manual). Unchanged from today. | unchanged |
| `/spec-tests-first:ship` | Inline commit/push/PR + merge. SDD-aware commit message and PR body. | **simplified** |

Helpers (unchanged in role, updated for the new cycle):
- `/spec-tests-first:update <feature>` — amend spec, mark changed ACs `stale`, no test regeneration (done by `/spec-tests-first:build`'s RGR loop).
- `/spec-tests-first:run <feature>` — orchestrator chains the 6 phases with checkpoints.
- `/spec-tests-first:status [<feature>]` — read-only inspector, adds `Latest review:` and findings counts.

## Section 1 — Architecture

### Agents

Three review agents are **copied into this plugin**, not depended on at install time:

```
plugins/spec-tests-first/agents/
├── test-runner.md             (existing — Haiku, Bash-only, JSON contract)
├── code-quality-reviewer.md   (NEW — copied from review-code, Sonnet)
├── security-reviewer.md       (NEW — copied from review-code, Sonnet)
└── code-reporter.md           (NEW — copied + extended: finding-id + Status column)
```

Reason for copying rather than depending: `/spec-tests-first:ship` historically declared external deps (`commit-commands`, `code-review`) and they were a friction point at install. Three small read-only agents are easy to keep in sync if `review-code` evolves; the alternative — adding a third install requirement — is a worse user experience.

### External dependencies

**None.** Removed:

| Previously | Why removed |
|---|---|
| `commit-commands@claude-plugins-official` | Inlined in `/spec-tests-first:ship` — an SDD-aware commit message/PR body (AC-IDs, validation summary, review status) is more useful than a generic one, and inlining is ~40 lines of Bash. |
| `code-review@claude-plugins-official` | Redundant with `/spec-tests-first:review` (runs the same kind of checks, after the fact, with merge already imminent). |

### System binaries

- `git` — used by `/spec-tests-first:build` (stash, branch, atomic commits per fix), `/spec-tests-first:fix` (per-fix commits, revert), and `/spec-tests-first:ship` (commit, push, branch detection).
- `gh` (GitHub CLI), authenticated — used by `/spec-tests-first:ship` only when a remote is configured (push, PR, merge).
- Project test runners — resolved from `CLAUDE.md ## Test commands` (and `## Test layout`); plugin remains language-agnostic.

## Section 2 — `/spec-tests-first:build` per-AC RGR engine

### Pre-checks

1. Spec exists at `docs/specs/<feature>/spec.md`.
2. Git initialized (offer `git init` if not; option to proceed without safety rails).
3. `.gitignore` scaffolded for the detected language (carry-over from v1).
4. Working tree clean — stash WIP if dirty under an SDD-named stash, then checkout `feature/<feature>` branch.

### Step 1 — Gather context

Read in order:
1. `docs/specs/<feature>/spec.md` — note every AC-ID and its discriminating signal (for error-path ACs).
2. `docs/specs/<feature>/spec-status.md` — if it exists (update cycle), identify `stale` ACs. Only iterate `stale` + `not-started` ACs.
3. `docs/codebase-map.md` — project-wide map; find existing modules to avoid duplication.
4. Existing test files for this feature (if update cycle).

### Step 2 — Resolve test command + framework

Read `CLAUDE.md ## Test commands` (multi-service-aware — see Section 3 below). If missing, auto-detect from project files (`pyproject.toml`, `package.json`, `Cargo.toml`, `go.mod`, `pom.xml`, etc.), confirm with the user via AskUserQuestion, and save the section to `CLAUDE.md`.

### Step 3 — Resolve test layout profile

Read `CLAUDE.md ## Test layout` (also multi-service-aware). If missing, derive a candidate profile from the test command and confirm with the user. Save to `CLAUDE.md`.

Each profile has these fields:

```yaml
profile: <name>
tests_root: <where the test files live, with <feature> token>
files: <naming pattern for test files>
fixtures: <where shared setup/teardown / config lives>
feature_anchor: directory | file_prefix | co-located
```

**Built-in profiles** (cover ~95% of real-world stacks):

| Profile | tests_root | files | fixtures | anchor |
|---|---|---|---|---|
| `python-pytest` | `tests/<feature>/` | `test_<cluster>.py` | `tests/<feature>/conftest.py` | directory |
| `python-unittest` | `tests/<feature>/` | `test_<cluster>.py` | `tests/<feature>/__init__.py` | directory |
| `js-jest` | `tests/<feature>/` | `<cluster>.test.ts` | `tests/<feature>/setup.ts` | directory |
| `js-vitest` | `tests/<feature>/` | `<cluster>.test.ts` | `tests/<feature>/setup.ts` | directory |
| `react-jest-rtl` | `src/features/<feature>/__tests__/` | `<Component>.test.tsx` | `__tests__/setup.ts` | directory |
| `angular-jasmine` | co-located with source | `<thing>.spec.ts` | `src/app/<feature>/testing.ts` | co-located |
| `spring-boot-junit5` | `src/test/java/<pkg>/<feature>/` | `<Cluster>Test.java` (unit), `<Cluster>IT.java` (integration) | `<Feature>TestConfig.java` | directory |
| `go` | same dir as source | `<source>_test.go` | `<feature>_testing.go` | co-located |
| `rust` | `tests/<feature>/` for integration; `#[cfg(test)]` for unit | `<cluster>.rs` | `tests/common/mod.rs` | directory |
| `dotnet-xunit` | `<Project>.Tests/<Feature>/` | `<Cluster>Tests.cs` | `<Feature>TestBase.cs` | directory |
| `ruby-rspec` | `spec/<feature>/` | `<cluster>_spec.rb` | `spec/spec_helper.rb` | directory |
| `phpunit` | `tests/<feature>/` | `<Cluster>Test.php` | `tests/<feature>/TestCase.php` | directory |

**Unknown stack:** ask the user inline for the 4 profile fields, save as `profile: custom`. No plugin update needed.

### Step 4 — Plan checkpoint (non-trivial specs only)

If the spec has more than ~3 AC-IDs or touches more than ~3 files, present a short plan (files to create, AC order) before writing code. Wait for user confirmation. For trivial specs, skip.

### Step 5 — Scaffold once

- For profiles with `feature_anchor: directory` or `file_prefix`: create the `tests_root` directory (per profile, per service for monorepos) and the framework fixtures (`conftest.py` / `setup.ts` / `<Feature>TestConfig.java` / etc.).
- For profiles with `feature_anchor: co-located` (e.g., `angular-jasmine`, `go`): no directory is created here. The `tests_root` field is informational; each test file is written next to its source file during Step 6's RGR loop. Shared testing helpers (`testing.ts`, `_testing.go`) are created lazily — on first AC that needs them.
- Initialize `spec-status.md` with the top-of-file metadata block (see Section 4 — Step 4 below) and all ACs marked `in-progress`.
- **No test bodies yet** — those come in Step 6.

### Step 6 — Per-AC red-green-refactor loop

For each `AC-ID` in dependency order:

```
┌────────────────────────────────────────────────────────────────────────┐
│  AC-N.M                                                                │
│                                                                        │
│  ┌─ RED ──────────────────────────────────────────────────────────────┐│
│  │ Write ONE test for AC-N.M in tests_root per profile.               ││
│  │ Embed AC-ID in test name (test_AC_1_1_*, AC-1.1 in describe).      ││
│  │ Use discriminating signal from spec for error-path ACs (specific   ││
│  │ stderr substring / exception class / status code — NEVER exit code ││
│  │ alone).                                                            ││
│  │                                                                    ││
│  │ Dispatch test-runner subagent (test_command, test_path, expected_  ││
│  │ ac_ids = [AC-N.M]). Parse JSON.                                    ││
│  │  → AC in `failed` (expected message)         : proceed to GREEN     ││
│  │  → AC in `passed`                            : test too permissive ││
│  │       — tighten assertion, retry RED                               ││
│  │  → AC in `errored`                           : fix the test, retry ││
│  │  → runner_failed / json_parse_failure        : surface and STOP    ││
│  └────────────────────────────────────────────────────────────────────┘│
│                          │                                             │
│  ┌─ GREEN ───────────────▼────────────────────────────────────────────┐│
│  │ Write the MINIMAL production code to pass this test.               ││
│  │ Read codebase-map.md first — don't duplicate existing modules.     ││
│  │ Honor the discriminating signal verbatim.                          ││
│  │                                                                    ││
│  │ Dispatch test-runner (cap = 3 attempts per AC).                    ││
│  │  → AC in `passed`                            : proceed to REFACTOR  ││
│  │  → AC in `failed`                            : fix and retry        ││
│  │  → cap-hit (3 same-cluster attempts)         : mark AC fail, STOP  ││
│  │  → same failure twice in a row, no progress  : STOP early           ││
│  └────────────────────────────────────────────────────────────────────┘│
│                          │                                             │
│  ┌─ REFACTOR (optional) ─▼────────────────────────────────────────────┐│
│  │ Clean up if helpful (remove duplication, improve names, extract    ││
│  │ helpers). NO new behavior. Re-run test-runner to confirm still     ││
│  │ green.                                                             ││
│  └────────────────────────────────────────────────────────────────────┘│
│                          │                                             │
│  ┌─ REGRESSION CHECK ────▼────────────────────────────────────────────┐│
│  │ Dispatch test-runner against the full feature suite (all ACs       ││
│  │ implemented so far).                                               ││
│  │  → all previously-pass ACs still pass        : mark AC-N.M = pass   ││
│  │       in spec-status.md, next AC                                   ││
│  │  → regression                                : mark AC-N.M blocked, ││
│  │       ask user (continue with next, or stop and investigate)       ││
│  └────────────────────────────────────────────────────────────────────┘│
└────────────────────────────────────────────────────────────────────────┘
```

**Checkpoint between ACs:** silent for ≤3 ACs; pause at **capability cluster boundaries** (e.g., between `AC-1.*` and `AC-2.*`) for bigger specs.

**Cap on retries:** **per AC**, not feature-wide. A 5-AC feature can have up to 15 attempts total, but each AC fails fast and gets attention individually.

### Step 7 — End-of-build artifacts

- Final `spec-status.md` (all ACs `pass` / `fail` / `blocked`).
- Merge into `docs/codebase-map.md` (project-wide; preserve other specs' entries).
- **Single commit at end of build** (if git in use): `git add -A && git commit -m "build(<feature>): implement spec — N ACs pass"`. Captures the full feature implementation as one commit on the feature branch. `/spec-tests-first:fix` then adds incremental commits on top; `/spec-tests-first:ship` only handles whatever (usually nothing) is uncommitted at ship time.

### Key invariants

| Invariant | Enforcement |
|---|---|
| No production code without a failing test first | RED step always runs `test-runner` and asserts AC is in `failed` before GREEN. |
| Watch the RED happen | The subagent returns JSON; `/spec-tests-first:build` verifies, never trusts "it should fail". |
| Test passes immediately = test is wrong | If RED returns `passed` for the new test, build flags "too permissive", does NOT proceed to GREEN. |
| One behavior per test | Each AC = at least one test, sometimes multiple (natural sub-cases), but each test asserts one behavior. |
| No weakening of failing tests | If GREEN fails 3 times, surface the issue; never edit the test to make it pass. |

## Section 3 — Monorepo handling

Many real repos are monorepos (frontend + backend in one tree). STF should work at the repo root and route each AC to the right test config.

### CLAUDE.md format — multi-service

```markdown
## Test commands

### service: frontend
| name | command       | default |
|------|---------------|---------|
| unit | `npm test`    | yes     |

### service: backend
| name | command       | default |
|------|---------------|---------|
| unit | `mvn test`    | yes     |

## Test layout

### service: frontend
profile: react-jest-rtl
tests_root: frontend/src/features/<feature>/__tests__/
files: <Component>.test.tsx
fixtures: __tests__/setup.ts
feature_anchor: directory

### service: backend
profile: spring-boot-junit5
tests_root: backend/src/test/java/com/example/<feature>/
files: <Cluster>Test.java
fixtures: <Feature>TestConfig.java
feature_anchor: directory
```

Single-service repos omit the `### service: <name>` headings — the format degrades to today's single-block layout. No breakage.

### Detection on first `/spec-tests-first:spec` in a repo

Scan for stack manifests in **subdirectories** (not just root). If multiple service manifests are found, prompt:

> Detected monorepo with services: **frontend** (node), **backend** (jvm).
> - (a, recommended) Configure multi-service — each can have its own profile + test command
> - (b) Treat as single-service — pick one to use STF on; ignore the others
> - (c) Cancel — I'll restructure first

On (a): walk per service through profile detection, save the multi-block CLAUDE.md.

### Spec.md additions for multi-service

Add a `Services:` line near the top (only when multi-service):

```markdown
# Feature: login-flow

Services: backend, frontend
```

AC sections can carry a service tag at the section header level (preferred) or inline:

```markdown
## Acceptance Criteria

### Backend (service: backend)
- AC-1.1: POST /api/login returns 200 with JWT on valid credentials
- AC-1.2: POST /api/login returns 401 + body `{"error":"invalid credentials"}` on bad password

### Frontend (service: frontend)
- AC-2.1: Login form submits credentials and stores returned JWT in localStorage
- AC-2.2: On 401 response, form shows "Invalid credentials" inline
```

`/spec-tests-first:build` reads the service for each AC, looks up that service's profile in `CLAUDE.md`, writes the test in the right `tests_root`, and uses that service's test command.

### Codebase-map for monorepos

Stays project-wide at the repo root. Adopts the optional service grouping pattern:

```markdown
## Server (service: backend)

| File | Role |
|------|------|
| `backend/src/main/java/com/example/auth/LoginCtrl.java` | POST /api/login handler |

## Client (service: frontend)

| File | Role |
|------|------|
| `frontend/src/features/login/LoginForm.tsx` | Login form component |
```

## Section 4 — `/spec-tests-first:review` (new phase)

Pure read-only review. Dispatches the two reviewer agents in parallel, aggregates via the reporter.

### Pre-checks

1. `spec-status.md` shows every AC = `pass`. (Build must be complete.)
2. The feature's test suite still green — a single `test-runner` dispatch before reviewers fire.

### Step 1 — Identify scope

Build the file list:
- Files changed on the feature branch vs `main`: `git diff main...HEAD --name-only`
- Cross-referenced with `docs/codebase-map.md` entries added/touched in this build
- The feature's test files (per profile / per service)

Exclude: generated files (lockfiles, snapshots, minified bundles), `docs/specs/<feature>/*`, `.gitignore`.

If empty, stop with "no files to review for `$1`".

### Step 2 — Dispatch reviewers in parallel

Via the Agent tool:
- `code-quality-reviewer` with the file list
- `security-reviewer` with the file list

Both run with tools restricted to `Read, Grep, Glob, Bash`. Each produces a Markdown report.

Optional scanner integrations run inside each reviewer in check-only mode if the binary is present (`eslint`, `tsc --noEmit`, `ruff`, `gitleaks`, `semgrep`, `npm audit`, `pip-audit`, `trivy`, etc.). Missing tools are logged and skipped.

### Step 3 — Aggregate

Dispatch `code-reporter` with both outputs. It writes:

```
./reports/code-review_<feature>_<YYYY-MM-DD_HHMMSS>.md
```

The report contains:

- Executive Summary (3–5 sentences)
- Combined Severity Dashboard (Critical / High / Medium / Low / Info)
- Top Priorities (5–7 consolidated items)
- **Status column on every finding, initialized to `pending`** (new — `/spec-tests-first:fix` mutates it)
- Security Findings (verbatim `# Security Review` block)
- Code Quality Findings (verbatim `# Code Quality Review` block)
- Metadata (timestamp, feature, scope, git short SHA, model names, total findings, output path)

Each finding gets a stable **`finding-id`** (e.g., `SEC-001`, `QUA-014`) embedded by the reporter, so `/spec-tests-first:fix` can address them by ID across multiple sessions without ambiguity.

### Step 4 — Brief summary + next-step pointer

Echo to user:

```
Review complete for `<feature>`.
  Report: <absolute path>
  Findings: 0 critical, 2 high, 5 medium, 3 low

Next: /spec-tests-first:fix <feature>   — walk findings one-by-one with revert-on-regression
   or /spec-tests-first:validate <feature> — proceed without fixing (you can /spec-tests-first:fix later)
```

**`Latest review:` pointer placement in `spec-status.md`:** added as a top-of-file metadata block, between the title and the AC table:

```markdown
# spec-status: <feature>

Last updated: <YYYY-MM-DD>
Latest review: <absolute path to most recent report>

## Status per Acceptance Criterion

| AC-ID | Status | Notes |
|---|---|---|
...
```

The pointer is overwritten every time `/spec-tests-first:review` (or `/spec-tests-first:fix`'s optional re-review) produces a new report.

### Design invariants

- Read-only by tool grant. Only `code-reporter` has `Write`, and only for the report file.
- No verdict gating — `/spec-tests-first:review` always reports; gating lives in `/spec-tests-first:fix` and in `/spec-tests-first:run`.
- The report path goes into `spec-status.md` as a one-line `Latest review: <path>` so `/spec-tests-first:fix` can locate the most recent report deterministically.
- Every invocation produces a **fresh timestamped report**. Old reports stay on disk for history.

## Section 5 — `/spec-tests-first:fix` (new phase)

Interactive fix loop. Reads the latest report, walks findings one-by-one, applies edits, revert-on-regression, mutates report in-place.

### Pre-checks

1. `spec-status.md` shows every AC = `pass`.
2. A review report exists. Resolve via `spec-status.md`'s `Latest review:` pointer; fall back to globbing `./reports/code-review_<feature>_*.md` and picking the most recent.
3. Working tree clean (stash WIP onto an SDD-named stash if dirty).

### Step 1 — Parse the report

Extract every finding by `finding-id`:

| field | source |
|---|---|
| `id` | reporter (e.g., `SEC-001`) |
| `severity` | Critical / High / Medium / Low / Info |
| `title` | one-liner problem |
| `file:line` | location |
| `why` | reviewer's reasoning |
| `suggested_fix` | concrete change |
| `status` | `pending` / `fixed` / `skipped` / `deferred` (may already be set from a previous session) |

Findings already `fixed` or `skipped` are not re-prompted — `/spec-tests-first:fix` is resumable.

### Step 2 — Build the queue

Order: severity descending (Critical → High → Medium → Low → Info), then by file inside each bucket. Skip already-handled findings.

### Step 3 — Walk the queue, one finding at a time

For each pending finding:

```
[SEC-001] [Critical]  backend/src/main/java/com/ex/auth/LoginCtrl.java:47
Title: SQL injection via String concatenation in user lookup
Why:   ...
Suggested fix: ...

(f) Fix — apply the suggested fix
(c) Custom — describe a different fix and I'll apply it
(s) Skip — mark as fixed without editing (already done / false positive)
(d) Defer — leave pending, revisit later
(q) Quit — keep progress so far
```

For **Critical findings**, `(s) Skip` is gated: a follow-up AskUserQuestion forces a one-line justification, which is recorded in the report.

### Step 4 — Apply with revert-on-regression

When the user picks `f` or `c`:

```
1. Edit/Write the change (scope to the finding's file and strictly related files).
2. Dispatch test-runner against the feature's tests (per profile / per service).
3. Parse JSON.
   ├─ all previously-pass ACs still pass → success
   │      ├─ status[finding-id] = fixed in report (in-place)
   │      ├─ atomic git commit: "fix(sdd:review): SEC-001 — <title>"
   │      └─ continue to next finding
   ├─ previously-pass AC now fails → fix broke something
   │      ├─ git checkout <file>  (revert just this edit)
   │      ├─ status[finding-id] = deferred, note "broke AC-X.Y, reverted"
   │      ├─ show user the regression
   │      └─ ask: continue with next finding or quit? (y/n)
   └─ test-runner errored → status = deferred, note error, continue
```

### Step 5 — Mutate the report in place

The reporter wrote a `Status` column initialized to `pending` for every finding. `/spec-tests-first:fix` updates each row as it goes — same file, no shadow copy.

Append a single `## Fix log` section at the bottom (one row per action):

```markdown
## Fix log

| time                | finding | action | result   | commit  | note                       |
|---------------------|---------|--------|----------|---------|----------------------------|
| 2026-05-11 14:23:01 | SEC-001 | fix    | success  | abc123  | parameterized query        |
| 2026-05-11 14:24:55 | QUA-007 | fix    | reverted | -       | broke AC-2.1               |
| 2026-05-11 14:25:30 | QUA-014 | skip   | -        | -       | false positive             |
```

Re-runs of `/spec-tests-first:fix` append more rows; never rewrite history.

### Step 6 — Critical gate

After the queue is exhausted (or user `q`uits):

- **Any Critical finding still `pending` or `deferred`** → stop with a clear list. User must address them or use AskUserQuestion to override with an explicit reason ("override critical: <reason>"), recorded in the report.
- **All Critical handled** (`fixed` or `skipped` with reason) → continue to Step 7.

### Step 7 — Re-run `/spec-tests-first:review`? (recommended)

```
> Re-run /spec-tests-first:review on the fixed code to confirm a clean state?
> - (a, recommended) Yes — produce a fresh report; if clean, continue to /spec-tests-first:validate
> - (b) No — trust the per-fix regression checks; continue to /spec-tests-first:validate now
> - (c) Stop here — I'll inspect the diff first
```

On (a): dispatch `/spec-tests-first:review` again, write a new timestamped report, update `spec-status.md`'s `Latest review:` pointer. If the new report has ANY Critical findings, return to `/spec-tests-first:fix` loop on the new report. Otherwise output next-step pointer to `/spec-tests-first:validate`.

On (b): output next-step pointer to `/spec-tests-first:validate`.

On (c): stop with current state preserved.

This makes the review/fix loop genuinely iterative: `review → fix → review → fix → ... → clean → validate`. The natural failure mode (infinite loop where each fix introduces a new finding) is bounded by the `(c) Stop here` option.

### Edge cases

| Case | Behavior |
|---|---|
| No report exists | Stop. "Run `/spec-tests-first:review $1` first." |
| Suggested-fix references code that's already changed (drift) | Edit fails or applies incorrectly → mark `deferred`, note "code drifted; re-run `/spec-tests-first:review`". |
| Multiple reports exist | Always pick the newest. User can override: `/spec-tests-first:fix $1 --report <path>`. |
| Phase invoked during `fail` / `blocked` build state | Stop. "Finish `/spec-tests-first:build $1` first." |

### Resumability

The report is the single source of truth and is mutated in place. Calling `/spec-tests-first:fix $1` a second time picks up exactly where it left off. No state lives in memory between sessions.

## Section 6 — `/spec-tests-first:ship` (simplified)

Inline commit/push/PR/merge with SDD-aware metadata.

### Pre-checks

1. `spec-status.md` shows every AC = `pass`.
2. **No `pending` or `deferred` Critical findings** in the latest review report. If found, stop with "address Critical findings in `/spec-tests-first:fix` first."
3. Pending git changes exist.
4. `gh` available and authenticated; `git remote -v` checked.

### Remote handling

No remote → ask: (a) commit locally only, (b) add remote then continue, (c) cancel.

### Step 1 — Stage and gather metadata

```
git add -A
git diff --cached --stat
```

Pull metadata from spec + status + report:

| Field | Source |
|---|---|
| Feature name | `$1` |
| Spec path | `docs/specs/$1/spec.md` |
| AC list with status | `spec-status.md` |
| Validation summary | `spec-status.md` post-validate section |
| Review report path | `spec-status.md` `Latest review:` line |
| Review summary | counts from report's `## Fix log` |
| Files touched | `git diff --cached --name-only` |

### Step 2 — Inline commit

Draft commit message from metadata using the project's existing style (scan `git log --oneline -20` to detect conventional vs free-form):

```
feat(<feature-or-area>): <short summary derived from spec goal>

Spec: docs/specs/<feature>/spec.md
ACs: AC-1.1 ✓ AC-1.2 ✓ AC-2.1 ✓ AC-2.2 ✓
Validation: 4/4 VS passed
Review: 8 fixed, 2 skipped, 0 deferred (0 critical, 0 high outstanding)

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
```

Use AskUserQuestion to show and confirm: (a) commit as-is, (b) edit message, (c) cancel.

### Step 3 — Push and open PR

```
git push -u origin feature/<feature>
gh pr create --title "<spec-derived title>" --body "<SDD-aware body>"
```

The body includes Summary (from spec.md Goal), Acceptance Criteria checklist, Validation steps checklist, code review link, and a Test plan section. Capture PR URL.

### Step 4 — Merge with explicit permission

```
> All checks done. PR: <url>.
> Merge now?
> - (a) Yes — squash
> - (b) Yes — merge commit
> - (c) Yes — rebase
> - (d) No — leave PR open
```

Recommend a default by scanning `git log --oneline main -20`. On merge, run `gh pr merge` with the chosen flag.

### Step 5 — Post-merge

```
Shipped. Spec `<feature>` is complete.
  Merged: <PR url>
  Status: docs/specs/<feature>/spec-status.md (all pass)
  Review: ./reports/code-review_<feature>_<ts>.md (status preserved)
```

### What ship no longer does

- ~~Inline pre-commit code review~~ → handled by `/spec-tests-first:review`.
- ~~`commit-commands:commit` / `commit-commands:commit-push-pr`~~ → inlined.
- ~~PR-level `/code-review`~~ → redundant with `/spec-tests-first:review`.

## Section 7 — Helpers (`/spec-tests-first:run`, `/spec-tests-first:update`, `/spec-tests-first:status`)

### `/spec-tests-first:run <feature>` — orchestrator

Chains the 6 phases with explicit checkpoints. Resumability detection (skip already-done phases):

| Artifact present | Phase done | Default action |
|---|---|---|
| `docs/specs/<f>/spec.md` | Spec | offer to skip |
| `spec-status.md` shows all ACs `pass` | Build | offer to skip |
| `./reports/code-review_<f>_*.md` exists | Review | offer to skip (re-run optional) |
| All Critical findings in latest report = fixed/skipped | Fix | offer to skip |
| `spec-status.md` shows `## Validation` block with all pass | Validate | offer to skip |

Checkpoints:

```
spec done    → "Continue to build? (y / edit / stop)"
build done   → "Continue to review? (y / stop)"
review done  → "Continue to fix? (y / skip / stop)"   ← "skip" goes straight to validate
fix done     → re-review prompt → "Continue to validate? (y / stop)"
validate done → "Continue to ship? (y / stop)"
ship done    → final summary
```

### `/spec-tests-first:update <feature>` — iteration

1. Snapshot `spec.md` → `spec.md.prev`.
2. Re-open spec for edits (same template as `/spec-tests-first:spec`).
3. After save, diff `spec.md.prev` vs `spec.md`:
   - **Changed AC-IDs** → mark `stale` in `spec-status.md`.
   - **New AC-IDs** → mark `not-started`.
   - **Removed AC-IDs** → mark `removed` (keep row for audit).
4. Tell user: "Run `/spec-tests-first:build <feature>` — only `stale` and `not-started` ACs iterate via RGR."

`/spec-tests-first:update` does NOT regenerate tests itself. `/spec-tests-first:build`'s RGR loop handles it, scoped to the changed AC subset.

### `/spec-tests-first:status [<feature>]` — read-only inspector

Same behavior as today; adds two new fields when drilling into a feature:

- `Latest review: <path>` — pointer to the most recent review report.
- `Findings: <fixed>/<total>` — aggregate from the report's status column.

Aggregate view (no arg) gets a new column: `Critical Outstanding` so specs blocked at the review/fix stage are visible at a glance.

## Section 8 — File structure

### Plugin tree

```
plugins/spec-tests-first/
├── README.md                        (rewritten for v2)
├── agents/
│   ├── test-runner.md               (unchanged)
│   ├── code-quality-reviewer.md     (NEW — copied from review-code)
│   ├── security-reviewer.md         (NEW — copied)
│   └── code-reporter.md             (NEW — copied + extended: finding-id + Status column)
├── commands/
│   └── status.md                    (updated to show review/fix fields)
└── skills/
    ├── spec/SKILL.md                (unchanged)
    ├── build/SKILL.md               (rewritten — per-AC RGR, profile-aware, monorepo)
    ├── review/SKILL.md              (NEW)
    ├── fix/SKILL.md                 (NEW)
    ├── validate/SKILL.md            (unchanged)
    ├── ship/SKILL.md                (rewritten — inline commit/PR, no external deps)
    ├── update/SKILL.md              (simplified — no test regeneration)
    └── run/SKILL.md                 (updated for 6-phase cycle)
```

**`skills/tests/SKILL.md`:** kept as a thin deprecation shim. The file's body is replaced with a short message that prints when invoked: "Folded into `/spec-tests-first:build`. Run `/spec-tests-first:build $1` directly — it now handles test scaffolding + per-AC RGR." No work performed; user is redirected. The shim can be removed entirely in a later release once existing scripts/runs are unlikely to invoke `/spec-tests-first:tests`.

### User-project artifacts (in target projects, not the plugin)

```
project-root/
├── CLAUDE.md                        (## Test commands + ## Test layout, multi-service capable)
├── docs/
│   ├── codebase-map.md              (project-wide)
│   └── specs/
│       └── <feature>/
│           ├── spec.md
│           ├── spec.md.prev
│           └── spec-status.md       (+ Latest review: pointer)
├── reports/
│   └── code-review_<feature>_<ts>.md  (in-place-mutated by /spec-tests-first:fix)
└── tests/<feature>/                  (or per profile / per service:
                                        src/features/<feature>/__tests__/,
                                        src/test/java/<pkg>/<feature>/, etc.)
```

## Section 9 — Backward compatibility

| v1 artifact | v2 behavior |
|---|---|
| `docs/specs/<feature>/spec.md` | Works unchanged. |
| `docs/specs/<feature>/spec-status.md` | Read as-is; missing `Latest review:` field is fine (added next time `/spec-tests-first:review` runs). |
| `tests/<feature>/` directories | Continue to work for Python projects; first new `/spec-tests-first:build` may prompt to confirm profile. |
| `/spec-tests-first:tests` command invocation | Deprecation shim that prints: "Folded into `/spec-tests-first:build`. Run `/spec-tests-first:build $1` directly — it now handles test scaffolding + per-AC RGR." No work performed. |
| CLAUDE.md `## Test commands` single-block | Reads unchanged. Multi-service requires user to opt in via monorepo detection on first `/spec-tests-first:spec`. |

## Section 10 — Open questions and decisions log

Decisions made during brainstorming (in chronological order):

1. **Scope: all three threads in one design.** Code-review integration, code-review phase, TDD upgrade — one coherent v2.
2. **TDD model: per-AC red-green-refactor.** Tests written inside `/spec-tests-first:build`, one at a time.
3. **Code-review placement: new phase between build and validate.**
4. **Ship's inline review: dropped.** Replaced by `/spec-tests-first:review`.
5. **Fix model: separate phase (`/spec-tests-first:fix`), interactive, severity-then-file ordering, status column in-place, revert-on-regression.**
6. **Re-run `/spec-tests-first:review` at end of `/spec-tests-first:fix`: yes, recommended.** Default to (a).
7. **Tests phase: dropped entirely.** Folded into `/spec-tests-first:build`.
8. **External plugin deps: dropped both** (`code-review` and `commit-commands`). STF v2 fully self-contained.
9. **Unknown stack: ask inline, save as `profile: custom`.**
10. **Monorepo handling: yes — multi-service CLAUDE.md, `Services:` line in spec, per-AC service tagging.**

No open questions remain.

## Implementation notes (for the next phase — writing-plans)

The plan should organize work into vertical slices that each leave the plugin runnable:

1. **Test-runner JSON contract verification.** Confirm the existing `test-runner` subagent's JSON contract is sufficient for per-AC dispatch (single-AC `expected_ac_ids`, regression-check full-list).
2. **`/spec-tests-first:build` rewrite.** Biggest change. Implement RGR loop, profile system, monorepo handling, cluster-boundary checkpoints, per-AC cap.
3. **Drop `/spec-tests-first:tests`** and add deprecation shim.
4. **Copy review agents.** `code-quality-reviewer`, `security-reviewer`, `code-reporter` into `plugins/spec-tests-first/agents/`. Extend `code-reporter` to emit `finding-id` and `Status` column.
5. **Add `/spec-tests-first:review` skill.**
6. **Add `/spec-tests-first:fix` skill.** Includes revert-on-regression, atomic-commit-per-fix, report mutation, Critical gate, optional re-review.
7. **Rewrite `/spec-tests-first:ship`.** Inline commit/push/PR/merge, drop external deps.
8. **Update `/spec-tests-first:run`, `/spec-tests-first:update`, `/spec-tests-first:status`.** Adjust for 6-phase cycle and new artifacts.
9. **Update README.md** for the new cycle, profiles, monorepo support.
10. **Migration test.** Run the new STF end-to-end on one of the existing sample features to validate backward compatibility and v1→v2 migration story.

Each slice ends with `/spec-tests-first:run` (or its v2 equivalent) demonstrating the cycle still works on a known-good feature.
