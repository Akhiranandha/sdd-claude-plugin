---
name: tests
description: Phase 2 of the SDD cycle. Use when the user invokes /sdd:tests <feature> to generate failing tests (TDD) from the spec at docs/specs/<feature>/spec.md. Generates one test per AC-ID under tests/<feature>/, embeds the AC-ID in every test name for traceability, and enforces the SDD discriminating-assertion rule for negative-path tests (must assert on stderr substring / exception class / status — never exit code alone). Runs a pre-implementation baseline to surface false-pass tests before the build phase.
argument-hint: <feature-name>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion, Agent
---

# /sdd:tests — Phase 2: Generate Tests from Spec

You are running Phase 2 of the SDD cycle for feature **$1**. Output: `tests/$1/`.

## Pre-check

Read `docs/specs/$1/spec.md`. If it does not exist, stop and tell the user:

> No spec found at `docs/specs/$1/spec.md`. Run `/sdd:spec $1` first.

In **scoped mode** (when invoked by `/sdd:update`), you'll receive a list of changed `AC-IDs`. Generate tests **only** for those — do NOT touch tests for unchanged ACs.

## Step 1 — Read the spec

Parse these sections:

| Section | What you extract |
|---|---|
| Section 3 (Acceptance Criteria) | Primary anchors — one test per `AC-N.M` |
| Section 5 (Technical details) | Stack → test framework + imports |
| Section 6 (Out of scope) | Negative anchors — write tests asserting these features are absent |
| Section 7 (Edge cases / open questions) | Open questions become `TODO(OQ-N)` comments in tests, not assertions |

The spec **must** name a discriminating signal for every error path in Section 3 (e.g. exit 2 + stderr "amount must be positive"). If an AC describes a rejection without naming a signal, **stop and ask the user to amend the spec via `/sdd:update`** — do not generate a weak negative test.

## Step 2 — Resolve framework + test command

You need two things from this step: the **test framework** (drives test-file syntax in Step 4) and the **resolved test command** (used by the baseline run in Step 6). Both come from one source: `CLAUDE.md` under a `## Test commands` section. This is project-wide config — `/sdd:build` will reuse it without re-prompting.

### 2a — Read CLAUDE.md

Look for a heading matching `## Test commands` (case-insensitive). If found, parse the table:

```markdown
## Test commands

| name | command              | default |
|------|----------------------|---------|
| unit | `python -m pytest`   | yes     |
| e2e  | `playwright test`    | no      |
```

Pick a row:

- If spec.md Section 5 names a runner by name (e.g. "Test framework: e2e"), pick that row.
- Otherwise, pick the row with `default: yes`. If multiple are default, pick the first.

### 2b — If CLAUDE.md is missing the section

Auto-detect from project files:

- `pyproject.toml` or `setup.py` → `python -m pytest`
- `package.json` with a `scripts.test` entry → use that script verbatim (`npm test`); else detect a jest / vitest binary in `node_modules/.bin/`
- `Cargo.toml` → `cargo test`
- `go.mod` → `go test ./...`
- `pom.xml` → `mvn test`

If the spec's Section 5 names a framework, use that to bias the detection (e.g. spec says "pytest" → confirm `python -m pytest`).

Confirm with the user via AskUserQuestion:

> Detected test command: `<cmd>`. Save this to CLAUDE.md so future SDD phases reuse it?
> - **(a, recommended)** Yes, save and proceed
> - **(b)** Override — let me type a different command (then save that)
> - **(c)** Use it once, don't save (skip CLAUDE.md write)

On (a) / (b): write the `## Test commands` section to CLAUDE.md.

- If CLAUDE.md does NOT exist, create it containing **only** the `## Test commands` section. Do not stub other sections.
- If CLAUDE.md exists, append the `## Test commands` section without modifying any existing content.
- If the section already exists but the table is malformed, replace **only that section** — never touch surrounding content.

### 2c — Derive the framework

Inspect the resolved command and pick the framework for code generation:

- contains `pytest` → **Python / pytest** (use `subprocess.run` for CLI tests, `pytest.raises` for library code)
- contains `jest` / `vitest` → **Node / TypeScript** (use that runner + `supertest` for HTTP)
- contains `go test` → **Go** (standard `testing` package + `testify` if available)
- contains `mvn` / `gradle` → **Java** (JUnit 5 + Mockito; `@SpringBootTest` for integration)
- contains `cargo test` → **Rust** (standard `#[cfg(test)]`)

If you can't tell from the command, fall back to spec.md Section 5. If still ambiguous, ask. Wrong framework wastes everyone's time.

## Step 3 — Plan the layers

Layer choice is driven by what the spec describes:

- **Unit tests** — pure logic, validators, formatters. Mock all I/O.
- **Integration tests** — HTTP / DB / file-system / queue round-trips. Use real components or testcontainers; not mocks.
- **E2E tests** — full UI scenarios from User Stories. Cypress / Playwright / Selenium.

If the spec is CLI-only, integration tests via subprocess are usually enough; skip unit and E2E layers. If the spec is API-only, skip E2E. Don't write all three layers by default — write what the spec actually warrants.

## Step 4 — Write the tests

### Traceability

Every test name embeds its AC-ID in a parser-stable form. Examples:

```python
def test_AC_1_1_add_stores_record_with_default_date(...): ...
def test_AC_1_2_add_rejects_negative_amount_with_named_error(...): ...
```

```typescript
it("AC-2.2 list filters by category case-insensitive", () => { ... });
```

The format `AC_N_M` (or `AC-N.M` in describe strings) MUST appear in every test name so a grep on the test output can map back to the spec.

**Multiple tests per AC-ID are allowed and encouraged when an AC has natural sub-cases** (e.g. AC-2.1 might have both `test_AC_2_1_list_prints_all_rows_in_insertion_order` and `test_AC_2_1_list_empty_data_prints_nothing`). The `test-runner` subagent groups every test sharing an AC-ID under one entry and the AC counts as `pass` only when all such tests pass.

### Discriminating assertions for negative-path tests — the SDD rule

Negative-path tests (anything asserting "this is rejected", "this errors", "this fails validation") are easy to write *too permissively*. A test that only checks `exit code != 0` will pass against a CLI that doesn't exist, segfaults, or prints garbage. That is a false-positive trap.

**Every negative-path test MUST assert at least one of:**

- A specific stderr / output substring (e.g. `"amount must be positive" in result.stderr`).
- A specific exception class (e.g. `pytest.raises(ValidationError)`).
- A specific named status / error code (e.g. HTTP `422`, not just `>= 400`).

**Never assert only on exit code != 0 / status >= 400.** The spec's Section 3 names the signal — use it. If the spec is silent, ask first.

### Happy paths first, then error paths

For each capability cluster (e.g. `add`, `list`, `summary`):

1. Happy path AC (`AC-N.1` typically).
2. Each error/rejection AC (`AC-N.2` etc.) — using discriminating assertions.
3. Edge cases from Section 7 if testable.

### Out-of-scope as negative tests

When Section 6 lists something as out of scope, write a test asserting absence:

```python
# Out of scope (spec §6) — soft-delete not supported in v1
def test_no_delete_subcommand_exists(run_cli):
    result = run_cli("delete", "1")
    assert result.returncode != 0
    assert "invalid choice" in result.stderr.lower() or "no such command" in result.stderr.lower()
```

### Open questions → TODOs

If an AC depends on an open question from Section 7, write a `TODO(OQ-N.M)` placeholder rather than guessing.

## Step 5 — File layout

Always nest tests under **`tests/$1/`** (e.g. `tests/money-tracker/`), regardless of whether the project already has a top-level `tests/` directory. Per-spec test isolation matters because `/sdd:update` regenerates tests scoped to changed AC-IDs — co-mingling specs in a flat tree makes that scoping unsafe.

```
tests/$1/
├── conftest.py                # or fixtures.ts / TestDataBuilder.java — shared fixtures + CLI runner
├── test_<capability>.py       # one file per capability cluster (e.g. test_add.py, test_list.py)
└── ...
```

Group AC-IDs by capability cluster (the major number — `AC-1.*` together, `AC-2.*` together). One file per cluster. Don't dump all tests into one file. Print the resolved path so the user knows where tests landed.

## Step 6 — Pre-implementation baseline (mandatory, via test-runner subagent)

After writing the tests, dispatch the `test-runner` subagent via the Agent tool, with `subagent_type: test-runner` and a prompt body that includes:

- `test_command`: the resolved command from Step 2.
- `test_path`: `tests/$1/`.
- `expected_ac_ids`: the full list of AC-IDs you wrote tests for.

Parse the returned JSON (extract the first ```json fenced block from the response, ignoring any preamble or trailing prose — parse defensively). The expected baseline is:

- **Most ACs in `failed`** — there is no implementation yet. That's TDD.
- **Negative-path ACs ALSO in `failed`** — their discriminating assertions can't be satisfied by an empty / missing implementation.

**False-positive trap**: if a negative-path AC appears in `passed` against an empty / missing implementation, its assertion is too permissive. Flag it to the user and tighten it (more specific substring, narrower exception class, narrower status code) before declaring this phase done.

If `error` is `runner_failed` (binary missing, suite can't compile), surface `stdout_tail` and stop — don't ship broken tests.

Report the baseline as: `X passed / Y failed / Z errored / W missing` (counts of AC-IDs in each bucket from the JSON).

## Step 7 — Coverage summary

After the tests pass the baseline, output a short table mapping the spec to the tests:

```markdown
## Coverage summary

| AC-ID | Layer | Test | Status |
|---|---|---|---|
| AC-1.1 | integration | `tests/$1/test_add.py::test_AC_1_1_*` | ready |
| AC-1.2 | integration | `tests/$1/test_add.py::test_AC_1_2_*` | ready (discriminating) |
| OQ-1.1 | — | — | TODO — open question |
| Out-of-scope: delete | integration | `tests/$1/test_oos.py::test_no_delete_*` | absence-asserted |
```

## Checkpoint

After tests are written and the baseline reported, output:

> Tests generated for `$1` under `tests/$1/`. Baseline: X passing / Y failing / Z errored — failures are expected for TDD.
> Run `/sdd:build $1` to implement the spec.

## Rules

- **Every test embeds its AC-ID.**
- **Every negative test uses a discriminating assertion.** Never exit code alone.
- **Open questions get TODOs, not guessed assertions.**
- **Don't write tests for things the spec doesn't name.** No drive-by coverage.
- **Don't weaken assertions to make tests pass.** Tests are the contract — if they fail, the implementation is wrong, not the test.
- **Output goes under `tests/$1/`** — never reuse a flat top-level test dir, even if one exists.
