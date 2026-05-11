---
name: build
description: Phase 2 of the SDD cycle. Use when the user invokes /sdd:build <feature> to implement the spec at docs/specs/<feature>/spec.md using per-AC red-green-refactor. Detects test framework + layout profile (12 built-in stack profiles plus a `custom` fallback; multi-service-aware for monorepos via per-service blocks in CLAUDE.md), scaffolds the test root (or no-op for co-located profiles), iterates ACs one-by-one writing a single failing test, watching RED via test-runner, writing minimal implementation, watching GREEN, then a regression check across the feature suite. Cap=3 attempts per AC; commits the whole feature once at the end. /sdd:tests is now a deprecation shim — this skill owns test scaffolding + writing under per-AC RGR.
argument-hint: <feature-name>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion, Agent
---

# /sdd:build — Phase 2: Per-AC Red-Green-Refactor

You are running Phase 2 of the SDD cycle for feature **$1**. Inputs: `docs/specs/$1/spec.md`. Outputs: working code + tests under the profile's `tests_root` + `docs/specs/$1/spec-status.md` + updated `docs/codebase-map.md` + one feature-level commit at the end.

## Pre-checks

1. **Spec exists?** `docs/specs/$1/spec.md` must exist. If not, stop:

   > No spec found at `docs/specs/$1/spec.md`. Run `/sdd:spec $1` first.

2. **Git initialized?** Run `git rev-parse --is-inside-work-tree` (silently). If NOT a git repo, AskUserQuestion:

   > This isn't a git repo. SDD's safety rails (stash, branch, atomic fix commits, end-of-build commit) need git. Choose:
   > - **(a, recommended)** Run `git init` and proceed
   > - **(b)** Proceed without git (no stash / branch / rollback / end-commit)
   > - **(c)** Cancel

   On (a): `git init`, continue. On (b): warn and proceed without safety rails. On (c): stop.

3. **`.gitignore` scaffolded?** If the project has no `.gitignore`, scaffold a minimal one based on the language detected from the spec's Section 5 ("Technical details") or project manifests. This is preventative — avoids artifact leaks at `/sdd:ship` time.

   - **Python** → `__pycache__/`, `*.pyc`, `*.pyo`, `.pytest_cache/`, `.venv/`, `venv/`, plus any runtime data file named in the spec (e.g. `data.json`, `*.db`).
   - **Node / TypeScript** → `node_modules/`, `dist/`, `build/`, `.env`, `coverage/`.
   - **Java (Maven/Gradle)** → `target/`, `build/`, `*.class`, `.gradle/`, `.idea/`.
   - **Go** → compiled binaries, `vendor/` (if used).
   - **Rust** → `target/`, `Cargo.lock` (only for libraries, not binaries — ask if unsure).
   - **Other** → ask the user.

   Always exclude the spec's named runtime data file (parse it out of "Technical details" / "Storage").

4. **Git stash + branch (if git is in use).**
   - `git status --porcelain`. If dirty: `git stash push -u -m "sdd:build pre-start stash for $1"`. Tell the user.
   - Create/checkout the feature branch: `git checkout -b feature/$1` (or `git checkout feature/$1` if it already exists).

## Step 1 — Gather context

Read in this order:

1. **`docs/specs/$1/spec.md`** — read fully. Note every `AC-ID` from Section 3 and the discriminating signal named in each error-path AC (the spec REQUIRES this; if any error-path AC lacks one, stop and tell the user to amend the spec via `/sdd:update $1`). If the spec has a `Services:` line at the top, this is a multi-service feature — also parse the per-section `(service: <name>)` tags or inline AC-level tags.
2. **`docs/specs/$1/spec-status.md`** — if it exists, read which `AC-ID`s are already `pass` vs `stale` vs `not-started`. In an `/sdd:update` cycle, only `stale` and `not-started` ACs need iteration; `pass` ACs are already implemented and their tests remain untouched.
3. **`docs/codebase-map.md`** — the project-wide map. Find existing modules so you don't duplicate.

After reading, you should be able to state in one sentence what each AC needs and where it goes. If you can't, you don't have enough context yet — read related code or ask.

## Step 2 — Resolve test command(s)

The project's test command lives in **`CLAUDE.md`** under a `## Test commands` section. Two shapes:

### Single-service shape

```markdown
## Test commands

| name | command            | default |
|------|--------------------|---------|
| unit | `python -m pytest` | yes     |
| e2e  | `playwright test`  | no      |
```

Pick a row:
- If spec.md Section 5 names a runner by name (e.g. "Test framework: e2e"), pick that row.
- Otherwise, pick `default: yes`. If multiple are default, pick the first.

### Multi-service shape

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
```

For each service detected in `spec.md`'s `Services:` line, pick its default-yes row. Cache as a `service → command` map.

### If `## Test commands` is missing

Auto-detect from project manifests, prompting the user:

- `pyproject.toml` / `setup.py` → `python -m pytest`
- `package.json` with a `scripts.test` entry → use that script (`npm test`); else detect a jest / vitest binary in `node_modules/.bin/`
- `Cargo.toml` → `cargo test`
- `go.mod` → `go test ./...`
- `pom.xml` → `mvn test`
- `build.gradle` / `build.gradle.kts` → `./gradlew test`
- `*.csproj` → `dotnet test`
- `Gemfile` with `rspec` → `bundle exec rspec`
- `composer.json` with `phpunit` → `vendor/bin/phpunit`

For monorepos (manifests in multiple subdirectories), `/sdd:spec` should have already prompted the user to configure multi-service. If they declined and chose single-service, only one manifest matters here. Otherwise iterate per service.

Confirm via AskUserQuestion:

> Detected test command(s):
>   <name>: `<cmd>`  (service: <svc> — if multi-service)
> Save to `CLAUDE.md` so future SDD phases reuse?
> - **(a, recommended)** Yes — save and proceed
> - **(b)** Override — let me type a different command (then save)
> - **(c)** Use once, don't save

On (a)/(b): write the `## Test commands` section to `CLAUDE.md`:
- If `CLAUDE.md` does NOT exist, create it with only this section.
- If it exists, append the section. Never modify other sections.
- If the section exists but is malformed, replace ONLY that section.

If the resolved command does NOT include a feature-scoped path (`python -m pytest` without `tests/$1/`, plain `npx jest`), append the profile's `tests_root` (resolved in Step 3) at use time. Do NOT modify commands that scope themselves via flags (e.g. `mvn -Dtest='*$1*' test`).

## Step 3 — Resolve test layout profile

Read `CLAUDE.md ## Test layout`. Two shapes mirroring `## Test commands`:

```markdown
## Test layout

profile: python-pytest
tests_root: tests/<feature>/
files: test_<cluster>.py
fixtures: tests/<feature>/conftest.py
feature_anchor: directory
```

Multi-service uses the same `### service: <name>` subheading pattern.

### Built-in profiles

| Profile | tests_root | files | fixtures | feature_anchor |
|---|---|---|---|---|
| `python-pytest` | `tests/<feature>/` | `test_<cluster>.py` | `tests/<feature>/conftest.py` | directory |
| `python-unittest` | `tests/<feature>/` | `test_<cluster>.py` | `tests/<feature>/__init__.py` | directory |
| `js-jest` | `tests/<feature>/` | `<cluster>.test.ts` | `tests/<feature>/setup.ts` | directory |
| `js-vitest` | `tests/<feature>/` | `<cluster>.test.ts` | `tests/<feature>/setup.ts` | directory |
| `react-jest-rtl` | `src/features/<feature>/__tests__/` | `<Component>.test.tsx` | `src/features/<feature>/__tests__/setup.ts` | directory |
| `angular-jasmine` | co-located with source | `<thing>.spec.ts` | `src/app/<feature>/testing.ts` | co-located |
| `spring-boot-junit5` | `src/test/java/<pkg>/<feature>/` | `<Cluster>Test.java` (unit), `<Cluster>IT.java` (integration) | `<Feature>TestConfig.java` | directory |
| `go` | same dir as source | `<source>_test.go` | `<feature>_testing.go` | co-located |
| `rust` | `tests/<feature>/` for integration; `#[cfg(test)]` for unit | `<cluster>.rs` | `tests/common/mod.rs` | directory |
| `dotnet-xunit` | `<Project>.Tests/<Feature>/` | `<Cluster>Tests.cs` | `<Feature>TestBase.cs` | directory |
| `ruby-rspec` | `spec/<feature>/` | `<cluster>_spec.rb` | `spec/spec_helper.rb` | directory |
| `phpunit` | `tests/<feature>/` | `<Cluster>Test.php` | `tests/<feature>/TestCase.php` | directory |

### Candidate selection

Derive from the resolved test command (Step 2):

| Command contains | Candidate profile | Ask follow-up? |
|---|---|---|
| `pytest` | `python-pytest` | — |
| `python -m unittest` | `python-unittest` | — |
| `jest` | `js-jest` | plain JS/TS or React? |
| `vitest` | `js-vitest` | plain or framework? |
| `ng test` / jasmine on `*.spec.ts` | `angular-jasmine` | — |
| `mvn test` / `gradle test` | `spring-boot-junit5` | plain JUnit if no Spring? |
| `go test` | `go` | — |
| `cargo test` | `rust` | — |
| `dotnet test` | `dotnet-xunit` | xunit / nunit / mstest? |
| `rspec` | `ruby-rspec` | — |
| `phpunit` | `phpunit` | — |

### Unknown stack → `custom`

If no candidate matches, AskUserQuestion to capture 4 fields inline:
1. `tests_root`: where do test files live? (e.g. `tests/<feature>/`)
2. `files`: file naming pattern? (e.g. `test_<cluster>.py`)
3. `fixtures`: shared setup/fixture file path? (e.g. `tests/<feature>/conftest.py`)
4. `feature_anchor`: `directory`, `file_prefix`, or `co-located`?

Save as `profile: custom` to `CLAUDE.md`.

### Confirm and save

Show the user two example file paths the chosen profile would produce for AC-1.1 and AC-2.1. AskUserQuestion to confirm. On confirm, write `## Test layout` to `CLAUDE.md` using the same append-don't-overwrite rule as Step 2.

For multi-service: repeat per service. Save each under `### service: <name>` subheadings.

## Step 4 — Plan checkpoint (non-trivial specs only)

If the spec has >3 AC-IDs or estimated >3 files to touch, present a short plan to the user before any test/code is written:

```
Plan for `$1`:
  Service: <name>  (only when multi-service)
  ACs (in dependency order):
    AC-1.1, AC-1.2, AC-2.1, AC-2.2, AC-3.1, AC-3.2
  Files to create:
    <path>: <one-line role>
    <path>: <one-line role>
  Estimated time per AC: ~2-5 min
```

5–10 lines max. AskUserQuestion to confirm / redirect / stop.

For trivial specs (≤3 ACs), skip this checkpoint silently.

## Step 5 — Scaffold once

### 5a — For profiles with `feature_anchor: directory` or `file_prefix`

Create the `tests_root` directory (per service in monorepos):

```bash
mkdir -p <tests_root>
```

Create framework fixtures by writing the `fixtures` file with the minimum needed to make tests importable. Examples:

- pytest: `conftest.py` with shared fixtures (CLI runner if CLI-tested, DB session if integration). Empty file if nothing yet.
- Jest/Vitest: `setup.ts` with `import '@testing-library/jest-dom'` or whatever the project pattern is.
- JUnit 5: `<Feature>TestConfig.java` with `@TestConfiguration` and `@Bean` overrides as needed.

Don't over-scaffold — fixtures grow during Step 6 if real ACs need them.

### 5b — For profiles with `feature_anchor: co-located`

Do NOT create directories here. Tests will be written next to source files during Step 6. The profile's `tests_root` field is informational. Shared helpers (`testing.ts`, `_testing.go`) are created lazily on the first AC that needs them.

### 5c — Initialize spec-status.md

Open or create `docs/specs/$1/spec-status.md`:

- **If it does not exist**, create it with this template:

  ```markdown
  # spec-status: $1

  Last updated: <YYYY-MM-DD>
  Latest review: (none yet)

  ## Status per Acceptance Criterion

  | AC-ID | Status | Notes |
  |---|---|---|
  | AC-1.1 | in-progress | (auto from /sdd:build) |
  | AC-1.2 | in-progress |  |
  ...
  ```

  The `Latest review:` line is initialized to `(none yet)` — `/sdd:review` updates it later.

- **If it exists** (e.g. `/sdd:update` cycle), set every `stale` and `not-started` AC-ID to `in-progress`. Leave `pass` ACs alone. Keep the existing `Latest review:` line if present.

## Step 6 — Per-AC red-green-refactor loop

For each AC-ID to iterate (all ACs for a new spec; only `in-progress` ACs after `/sdd:update`), in dependency order:

```
RED → GREEN → REFACTOR (optional) → REGRESSION CHECK → next AC
```

For multi-service: each AC has a service tag. Use that service's test command and `tests_root`.

### RED — Write ONE failing test

Write a single test for AC-N.M in the appropriate file (per profile):
- Embed the AC-ID in the test name (`test_AC_1_1_*`, `AC-1.1` in describe strings).
- For error-path ACs, use the discriminating signal from the spec verbatim (specific stderr substring / exception class / status code). NEVER assert only on `exit code != 0` or `status >= 400` — those are too permissive and the baseline will catch them.
- For positive-path ACs, assert on the documented behavior (return value, side effect, state change).

Dispatch the `test-runner` subagent with a **single-AC** input:

```
Agent[test-runner]
  subagent_type: test-runner
  prompt: |
    test_command: <resolved command for this service, with tests_root appended if needed>
    test_path: <path to the test file or its parent directory>
    expected_ac_ids: ["AC-N.M"]
```

Parse the JSON. Expected outcome: AC-N.M in `failed`.

| Actual | Action |
|---|---|
| AC in `failed` | RED verified — proceed to GREEN |
| AC in `passed` | Test is too permissive (passes against no implementation). Tighten the assertion (narrower substring, more specific exception class, exact status code) and re-run RED. Show user the change. |
| AC in `errored` | Likely import/syntax error in the test. Fix the test, re-run RED. |
| AC in `missing_ac_ids` | Test name doesn't embed the AC-ID correctly. Rename the test, re-run RED. |
| `error: runner_failed` / `json_parse_failure` | Surface `stdout_tail`, stop the whole phase. |

### GREEN — Minimal production code

Read `codebase-map.md` for existing modules — don't duplicate.

Write the simplest code that makes this AC's test pass. Honor the discriminating signal verbatim — if the test expects stderr to contain `"amount must be positive"`, that exact substring must appear.

Dispatch `test-runner` with the same inputs as RED.

| Outcome | Action |
|---|---|
| AC in `passed` | GREEN verified — proceed to REFACTOR |
| AC in `failed` | Code doesn't satisfy assertion. Analyze the JSON's `failed[].message`, fix the code (NOT the test), retry. **Cap = 3 attempts per AC.** |
| Same failure twice in a row (identical `failed[].message`) | Fix isn't fixing it. STOP early, don't waste the third attempt. |
| Cap hit (3 attempts, still failing) | Mark AC `fail` in `spec-status.md` with a summary of the 3 attempts in `## Test-fix loop log`. AskUserQuestion: (a) keep branch for debugging, (b) revert this AC's changes and continue with next, (c) stop the whole build. Default: (a). |

**Never** weaken, skip, disable, or delete the failing test. **Never** silently retry past cap=3.

### REFACTOR — Clean up (optional, post-green)

After GREEN passes, optionally clean up — remove duplication, improve names, extract helpers. NO new behavior.

If you refactor, re-dispatch `test-runner` to confirm the test is still green. If REFACTOR breaks the test → revert just the refactor, keep the GREEN code.

### REGRESSION CHECK — Full feature suite

Dispatch `test-runner` against the full feature test suite (all ACs implemented so far), with the FULL `expected_ac_ids` list:

```
Agent[test-runner]
  subagent_type: test-runner
  prompt: |
    test_command: <command>
    test_path: <feature's tests_root or co-located scope>
    expected_ac_ids: [<every AC up to and including this one>]
```

Outcomes:

| Outcome | Action |
|---|---|
| All implemented ACs in `passed` | Mark this AC `pass` in `spec-status.md`. Next AC. |
| Previously-pass AC now in `failed` / `errored` | This AC's implementation broke a prior AC. Revert just this AC's diff (`git checkout` files touched by this AC's GREEN step; new files created by this AC: `rm` them). Mark this AC `blocked` with a note pointing at the regressed AC. AskUserQuestion: continue with next AC or stop? |

### Checkpoint between ACs (medium+ specs only)

- For ≤3 ACs: silent — go straight to next AC.
- For >3 ACs: at **capability cluster boundaries** (between `AC-1.*` and `AC-2.*`, between `AC-2.*` and `AC-3.*`, etc.), AskUserQuestion:

  > Cluster `AC-N.*` complete (all <count> ACs `pass`). Continue with cluster `AC-(N+1).*`?
  > - **(a, recommended)** Continue
  > - **(b)** Pause — show me the diff so far
  > - **(c)** Stop here

  On (b): print `git diff` summary, then re-ask. On (c): stop the loop, fall through to Step 7 with partial state.

## Step 7 — End-of-build artifacts (single commit)

After the loop completes (success OR cap-hit), write artifacts.

### 7a — Final `spec-status.md`

Update every AC's status to its final value (`pass` / `fail` / `blocked` / `stale`). Append a `## Test-fix loop log` section if any AC took >1 GREEN attempt (one entry per attempt with the JSON's `failed[].message` summary). Append a `## Deviations from spec` section if the implementation diverged from the spec text.

### 7b — Project-wide `docs/codebase-map.md`

This is **one file for the whole project**, not per-feature. Update it in place — never create `docs/specs/$1/codebase-map.md`.

Procedure:

1. Read existing `docs/codebase-map.md`. If absent, create with the template below.
2. Collect every source file this build touched (created or modified) + the spec's runtime data file (named in Section 5).
3. Merge rows:
   - File already has a row → update role description if this spec changed its purpose; otherwise leave it.
   - File is new → append a row.
   - Rows for files NOT touched by this build → preserve verbatim.
4. Append any new invariants under `## Key invariants`. Don't delete prior invariants unless they're now wrong — flag those with a one-line note on removal.

Fresh-file template:

```markdown
# codebase-map

Project-wide map of source files and their roles. Updated by `/sdd:build` after each spec.

| File | Role |
|------|------|
| `<path>` | <one-line role> |

## Key invariants

- <one-liner per non-obvious invariant>
```

Monorepos may group rows under `## Server`, `## Client`, `## Shared`, `## Tests` headers (each with its own table). Pick a structure and keep it consistent across builds.

### 7c — Single end-of-build commit

If git is in use:

```bash
git add -A
git commit -m "build($1): implement spec — <N> ACs pass"
```

Substitute `<N>` with the count of `pass` ACs. This captures the full feature implementation as a single commit on the feature branch. `/sdd:fix` will add atomic commits on top per finding fixed.

If git is not in use (pre-check 2 (b)), skip — no commit.

## Failure rollback prompt

If implementation hard-fails (cap-hit on multiple ACs, or unrecoverable error), AskUserQuestion:

> Implementation failed for `$1`. Choose:
> - **(a, recommended)** Keep the branch as-is for debugging
> - **(b)** Reset the branch to the pre-implementation commit (`git reset --hard <pre-impl-SHA>`)
> - **(c)** Delete the branch and restore the stash

Execute the user's choice. Never default to a destructive option.

## Checkpoint on success

When all target ACs pass, output:

```
Implementation complete for `$1`. <N> ACs pass.
  spec-status.md: docs/specs/$1/spec-status.md
  codebase-map.md: docs/codebase-map.md
  Commit: <short SHA>

Next: /sdd:review $1
```

## Guardrails

- **Artifact paths.** `spec-status.md` is per-feature at `docs/specs/$1/spec-status.md`. `codebase-map.md` is project-wide at `docs/codebase-map.md`. NEVER write a per-spec codebase-map.
- **Cap = 3 per AC.** Never feature-wide. No silent retries.
- **No weakening of failing tests.** Surface to the user instead.
- **No drive-by refactoring.** Scope changes to the spec.
- **Never** `git push`, `--force`, `--no-verify`, or `git reset --hard` in this flow (except the rollback prompt's option (b), which is explicitly user-chosen).
- **Always** dispatch `test-runner` for RED and GREEN verification — never trust a hand-eyeballed "should fail" / "should pass".
- **Always** atomic commit per build is ONE commit; `/sdd:fix` adds incremental commits on top.
- **Multi-service:** resolve service per AC; use the right command + profile per service. Don't co-mingle.
- **Co-located profiles:** test files live next to source; the test name's AC-ID embedding is how `/sdd:update` finds them later.
