---
name: spec
description: Phase 1 of the SDD cycle. Use when the user invokes /sdd:spec <feature> to interactively author or amend a feature spec at docs/specs/<feature>/spec.md. Produces the SDD 8-section template (Goal, Requirements, Acceptance Criteria with AC-N.M IDs, User stories, Technical details, Out of scope, Edge cases, Validation steps with VS-N IDs). Sub-capabilities of one feature go in one spec as separate AC-IDs — never multiple specs. Pauses at a checkpoint so the user can review before generating tests.
argument-hint: <feature-name>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion
---

# /sdd:spec — Phase 1: Write the Spec

You are running Phase 1 of the SDD cycle for feature **$1**. Output: `docs/specs/$1/spec.md`.

## Pre-check

If `docs/specs/$1/spec.md` already exists, ask the user via AskUserQuestion:

> A spec for `$1` already exists. Choose: **(a)** overwrite, **(b)** update via `/sdd:update $1`, **(c)** cancel.

Stop unless they choose (a).

## Multi-feature handling — the hard rule

**Sub-capabilities of one feature go in ONE spec as separate Acceptance Criteria — not multiple specs.**

If the user describes `add` / `list` / `summary` for a money tracker, that is **one** spec with multiple `AC-IDs` (e.g. `AC-1.1` for add, `AC-2.1` for list, `AC-3.1` for summary). It is NOT three specs.

Only ask "single spec or multiple?" if the user describes capabilities that don't share a domain model and could ship independently (e.g. a calendar app and a money tracker — those are two features). When in doubt, default to one spec — over-splitting is harder to recover from than under-splitting.

If — and only if — the user is describing multiple genuinely separate features, ask:

> You've described what sound like multiple features. Should I write **(a, recommended)** one spec per feature in its own `docs/specs/<feature>/` folder, implemented sequentially, or **(b)** one combined spec covering all of them?

## Step 1 — Orient

Quick context gather, in this order. Stop when you have enough:

1. **The user's prompt** — stack mentioned? domain hints? out-of-scope hints? Use them.
2. **`CLAUDE.md`** at the project root, if it exists — pick up stack / conventions.
3. **Existing specs** at `docs/specs/*/spec.md` — match their tone, but don't infer technical conventions from their content (those come from CLAUDE.md or the user).

## Step 2 — Ask, don't guess

If any of the following are unknown, ask **all** questions in one batch under a "Before I write the spec, a few quick questions:" header. Never assume:

- **Stack** — language, framework, datastore, runtime. Wrong stack means wrong tests.
- **Actor / role** — who uses this?
- **Integrations** — third-party services, queues, APIs?
- **Conventions** — error response shape, base URL, auth pattern (only ask if the feature involves them).
- **Out of scope** — adjacent things to explicitly exclude.

Skip questions only if the user's prompt + CLAUDE.md + existing specs cover every gap. When in doubt, ask.

## Step 3 — Write the spec

Use **exactly** this 8-section template, in this order. Save to `docs/specs/$1/spec.md`.

```markdown
# Spec: <feature>

## 1. Goal

One or two sentences: why this exists, what user problem it solves.

## 2. Requirements

- The system **must** ...
- The system **should** ...
- Users **can** ...

Capability statements using must / should / can. Plain language. NOT testable assertions — those go in Section 3.

## 3. Acceptance Criteria

Binary, testable assertions. Each MUST have an `AC-N.M` ID. Be concrete enough that a test author can write an assertion straight from the line.

For every error / rejection / failure path, **name the discriminating signal** the implementation must produce — a specific stderr substring, exception class, HTTP status, error code. The test phase requires this; vague error contracts produce false-pass tests.

- **AC-1.1:** ...
- **AC-1.2:** <bad input> is rejected with <specific signal — e.g. exit 2 + stderr substring "X is required">.
- **AC-2.1:** ...

Group AC-IDs by capability. The major number is the capability cluster (1 = add, 2 = list, etc.); the minor number is the case (1 = happy path, 2 = error, 3 = edge).

## 4. User stories

- As a [role], I want to [action], so that [benefit].

One per distinct user goal. Concrete.

## 5. Technical details

Fill in only what applies. Don't invent details that the user didn't state and CLAUDE.md doesn't supply.

- **Stack** — language / framework / datastore / runtime
- **Entry point / module** — where this code lives
- **Data model** — tables, schemas, file format
- **API surface** — endpoints / methods / request-response (only if the feature has an API)
- **Storage** — name the runtime data file or DB if the feature persists state (so /sdd:build can add it to .gitignore)
- **Test framework** — pytest / jest / etc., to lock the test phase
- **Integrations** — external services, queues, third-party APIs

## 6. Out of scope

- Explicit list of adjacent capabilities NOT in this spec
- Cross-reference related specs by feature name where relevant

## 7. Edge cases / open questions

- Tricky situations the implementation must handle
- Anything unresolved that needs a decision before or during implementation

## 8. Validation steps

Concrete steps to manually verify the feature works end-to-end. Each step MUST have a `VS-N` ID. Used by `/sdd:validate`. Mix automated (commands, scripts, API calls) and manual (UI checks).

- **VS-1** *(automated)*: <exact command to run> — <expected outcome>
- **VS-2** *(manual)*: <step a human performs> — <what they should see>
```

## Step 4 — Confirm and checkpoint

Print one short confirmation listing:

- The path written (`docs/specs/$1/spec.md`).
- The list of AC-IDs and VS-IDs you assigned.
- Any open questions still unresolved.

Then output the checkpoint message exactly:

> Spec written to `docs/specs/$1/spec.md`. Review it and tell me any changes you want.
> When you're happy, run `/sdd:tests $1` to generate tests.

DO NOT proceed to test generation in this skill — the user must explicitly trigger the next phase.

## Rules

- **Never implement code.** Only write the spec.
- **Never reuse an `AC-ID` or `VS-ID`** that already exists in the file (in update mode, append new IDs — don't renumber).
- **Never guess at stack or conventions** — ask in Step 2.
- **Keep it tight** — under 200 lines if possible. No padding, no repetition.
- **Every error path in Section 3 names a discriminating signal.** Vague error contracts ("an error is shown") produce false-pass tests.
- **Sub-capabilities of one feature = one spec, multiple AC-IDs.** Don't split.
