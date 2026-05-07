---
name: spec
description: Phase 1 of the SDD cycle. Use when the user invokes /spec-flow:spec <feature> to interactively author a feature spec at docs/specs/<feature>/spec.md (amendments to an existing spec go through /spec-flow:update instead). Produces the SDD 6-section template (Goal, Requirements, User Stories with US-N IDs and optional Done-when checks, Technical details, Out of scope, Edge cases). Sub-capabilities of one feature go in one spec as separate US-IDs — never multiple specs. Pauses at a checkpoint so the user can review before implementation.
argument-hint: <feature-name>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion
---

# /spec-flow:spec — Phase 1: Write the Spec

You are running Phase 1 of the SDD cycle for feature **$1**. Output: `docs/specs/$1/spec.md`.

## Pre-check

If `docs/specs/$1/spec.md` already exists, ask the user via AskUserQuestion:

> A spec for `$1` already exists. Choose: **(a)** overwrite, **(b)** update via `/spec-flow:update $1`, **(c)** cancel.

Stop unless they choose (a).

## Multi-feature handling — the hard rule

**Sub-capabilities of one feature go in ONE spec as separate User Stories — not multiple specs.**

If the user describes `add` / `list` / `summary` for a money tracker, that is **one** spec with multiple `US-N` IDs (e.g. `US-1` for add, `US-2` for list, `US-3` for summary). It is NOT three specs.

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

- **Stack** — language, framework, datastore, runtime.
- **Actor / role** — who uses this?
- **Integrations** — third-party services, queues, APIs?
- **Conventions** — error response shape, base URL, auth pattern (only ask if the feature involves them).
- **Out of scope** — adjacent things to explicitly exclude.

Skip questions only if the user's prompt + CLAUDE.md + existing specs cover every gap. When in doubt, ask.

## Step 3 — Write the spec

Use **exactly** this 6-section template, in this order. Save to `docs/specs/$1/spec.md`.

```markdown
# Spec: <feature>

## 1. Goal

One or two sentences: why this exists, what user problem it solves.

## 2. Requirements

- The system **must** ...
- The system **should** ...
- Users **can** ...

Capability statements using must / should / can. Plain language. High-level — the concrete user flows and acceptance live in Section 3.

## 3. User Stories

Each story has a stable `US-N` ID. `Done when:` blocks are optional but recommended — they are how `/spec-flow:validate` confirms the story works. Mix automated (commands, scripts, API calls) and manual (UI checks).

For error / rejection / failure stories, **name the discriminating signal** in the relevant Done-when check — a specific stderr substring, exception class, HTTP status, error code. Vague signals make validation unreliable.

- **US-1**: As a [role], I want to [action], so that [benefit].
  - Done when:
    - _(automated)_ `<exact command>` — <expected outcome>
    - _(manual)_ <step a human performs> — <what they should see>
- **US-2**: As a [role], I want to [action], so that [benefit].
  - Done when:
    - _(automated)_ `<command>` — exits 2, stderr contains "amount must be positive"
- **US-3**: As a [role], I want to [action], so that [benefit].
  _(no Done-when block — `/spec-flow:validate` will ask the user to manually confirm)_

One story per distinct user goal. Group related stories adjacently.

## 4. Technical details

Fill in only what applies. Don't invent details that the user didn't state and CLAUDE.md doesn't supply.

- **Stack** — language / framework / datastore / runtime
- **Entry point / module** — where this code lives
- **Data model** — tables, schemas, file format
- **API surface** — endpoints / methods / request-response (only if the feature has an API)
- **Storage** — name the runtime data file or DB if the feature persists state (so /spec-flow:build can add it to .gitignore)
- **Integrations** — external services, queues, third-party APIs

## 5. Out of scope

- Explicit list of adjacent capabilities NOT in this spec
- Cross-reference related specs by feature name where relevant

## 6. Edge cases / open questions

- Tricky situations the implementation must handle
- Anything unresolved that needs a decision before or during implementation
```

## Step 4 — Confirm and checkpoint

Print one short confirmation listing:

- The path written (`docs/specs/$1/spec.md`).
- The list of US-IDs you assigned, and which have Done-when blocks vs which will fall back to manual confirmation.
- Any open questions still unresolved.

Then output the checkpoint message exactly:

> Spec written to `docs/specs/$1/spec.md`. Review it and tell me any changes you want.
> When you're happy, run `/spec-flow:build $1` to implement.

DO NOT proceed to implementation in this skill — the user must explicitly trigger the next phase.

## Rules

- **Never implement code.** Only write the spec.
- **Never reuse a `US-N` ID** that already exists in the file (in update mode, append new IDs — don't renumber).
- **Never guess at stack or conventions** — ask in Step 2.
- **Keep it tight** — under 200 lines if possible. No padding, no repetition.
- **For error-path stories, name the discriminating signal in the Done-when check.** Vague signals ("an error is shown") make `/spec-flow:validate` unreliable.
- **Sub-capabilities of one feature = one spec, multiple US-IDs.** Don't split.
