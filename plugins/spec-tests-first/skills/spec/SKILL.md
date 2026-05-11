---
name: spec
description: Phase 1 of the SDD cycle. Use when the user invokes /sdd:spec <feature> to interactively author or amend a feature spec at docs/specs/<feature>/spec.md. On first invocation in a repo, detects monorepos and offers to configure multi-service (per-service test commands + layout profiles in CLAUDE.md). Produces the SDD 8-section template (Goal, Requirements, Acceptance Criteria with AC-N.M IDs, User stories, Technical details, Out of scope, Edge cases, Validation steps with VS-N IDs). For multi-service specs, AC clusters are grouped under per-service subsection headers. Sub-capabilities of one feature go in one spec as separate AC-IDs — never multiple specs. Pauses at a checkpoint so the user can review before /sdd:build begins per-AC red-green-refactor.
argument-hint: <feature-name>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion
---

# /sdd:spec — Phase 1: Write the Spec

**Announce at start:** Say to the user: "I'm using /sdd:spec to author `$1` interactively (8-section template, monorepo + scope check + explicit user approval at the end)." Then proceed.

You are running Phase 1 of the SDD cycle for feature **$1**. Output: `docs/specs/$1/spec.md` + a `## Phase progress` block in `docs/specs/$1/spec-status.md` with Phase 1 marked `done` only after the user explicitly approves.

## Iron Law

> **Don't mark Phase 1 = `done` until the user picks `(a) Approve` at the Step 4 gate. A spec the user hasn't read is a spec waiting to fail in `/sdd:build`.**

`/sdd:build`'s pre-check 1 enforces this — it won't run against a spec whose Phase 1 status isn't `done`. So skipping the gate here just produces a confusing "spec exists but build refuses to start" error downstream.

## Pre-check

If `docs/specs/$1/spec.md` already exists, ask the user via AskUserQuestion:

> A spec for `$1` already exists. Choose: **(a)** overwrite, **(b)** update via `/sdd:update $1`, **(c)** cancel.

Stop unless they choose (a).

## One-feature-per-spec — the hard rule

**Sub-capabilities of one feature go in ONE spec as separate Acceptance Criteria — not multiple specs.** Multiple genuinely independent features go in **separate specs**, implemented sequentially. The full split-vs-combine decision logic lives in Step 1.5 below — invoke it whenever the user's request spans capabilities that could ship independently.

## Step 0 — Monorepo detection (on first `/sdd:spec` only)

If `CLAUDE.md ## Test commands` already exists, skip this step — the multi-service decision was made on a prior invocation.

Otherwise, scan for stack manifests in **subdirectories** of the current working directory (use Glob for each):

- `package.json` — Node service
- `pom.xml` / `build.gradle` / `build.gradle.kts` — JVM service
- `Cargo.toml` — Rust service
- `go.mod` — Go service
- `pyproject.toml` / `setup.py` — Python service
- `Gemfile` — Ruby service
- `composer.json` — PHP service
- `*.csproj` — .NET service

A manifest at the **repo root** is single-service (or workspace manager) — not a monorepo signal. The monorepo signal is **two or more manifests in distinct subdirectories** (e.g. `frontend/package.json` and `backend/pom.xml`).

If multiple service manifests are found, AskUserQuestion:

> Detected monorepo with services:
>   - **<name1>** (<stack1>) at `<subdir1>/`
>   - **<name2>** (<stack2>) at `<subdir2>/`
>
> How should this spec target the services?
> - **(a, recommended)** Configure multi-service — this feature may touch any subset; each gets its own profile + test command in CLAUDE.md
> - **(b)** Single-service — pick one to use STF on; ignore the others
> - **(c)** Cancel — I'll restructure first

On (a): record the detected service names; the spec template below adds a `Services:` line and AC sections can carry service tags. Test commands and profiles get configured in `/sdd:build` on first run (one per service).

On (b): record the chosen service name; the spec stays in single-service shape (no `Services:` line, flat AC list).

On (c): stop.

If only one (or zero) manifest is found in subdirectories, this is not a monorepo — proceed to Step 1 without prompting. Single-service flow.

## Step 1 — Orient

Quick context gather, in this order. Stop when you have enough:

1. **The user's prompt** — stack mentioned? domain hints? out-of-scope hints? Use them.
2. **`CLAUDE.md`** at the project root, if it exists — pick up stack / conventions.
3. **Existing specs** at `docs/specs/*/spec.md` — match their tone, but don't infer technical conventions from their content (those come from CLAUDE.md or the user).

## Step 1.5 — Scope check (catch oversized specs BEFORE writing)

Before going further, evaluate whether the user's request is one feature or several. Prevention is much cheaper than splitting an already-written spec after `/sdd:build` and `/sdd:review` have run against it.

### Heuristic — what counts as "should split"?

A request is **one feature** (one spec, multiple AC clusters) when its capabilities:
- Share a domain model (same primary entity — e.g. "user signup" + "user profile editing" both touch the `users` table).
- Need to ship together to be useful (e.g. "API endpoint" + "client form that calls it").
- Belong to the same user goal expressed in the prompt.

A request is **multiple features** (separate specs, ideally sequential) when:
- The capabilities have **no shared domain model** (e.g. "auth flow" + "billing dashboard" — different entities, different tables, different teams could ship them).
- Either could ship **without the other** and still be useful.
- The user listed them as bullets / "and also" / "plus" — surface signal that they're parallel asks.

The 50% case (capabilities loosely related but each substantial): default to **split**. Combining is hard to undo; splitting later is harder. Spec files should stay focused — under ~200 lines with ≤8 AC clusters is a healthy ceiling.

### Detection + prompt

If the request looks like multiple features, AskUserQuestion before going to Step 2:

> Looking at what you described, this might be **N independent features**:
>   - **`<name1>`** — `<one-line goal>` (involves <entities/files>)
>   - **`<name2>`** — `<one-line goal>` (involves <entities/files>)
>
> They could ship independently — `<name1>` doesn't need `<name2>` to work, and vice versa.
>
> Choose:
> - **(a, recommended)** Write one spec per feature, sequentially. I'll start with `<name1>` (or you pick) and you can `/sdd:spec <name2>` after.
> - **(b)** Combine into one spec with the capabilities as separate AC clusters (`AC-1.*`, `AC-2.*`, ...). Better only if they share enough domain model.
> - **(c)** Let me clarify — they're actually coupled because <reason>. I'll describe the relationship and you'll decide.

### Outcomes

On **(a) Split**:
- AskUserQuestion which one to spec first (default: the one with the simplest interface, or the user's first mention if order was clear).
- Create a `## Related features` reference in this spec's Section 5 (Technical details) listing the other feature names and a one-line goal each. This is a forward-reference — `/sdd:spec <other-name>` later can do the inverse.
- Continue with Step 2 below for the chosen first feature only.

On **(b) Combine**:
- Continue with Step 2 below treating the request as one feature with multiple AC clusters.
- Warn the user: if the combined spec grows past ~200 lines, you'll suggest splitting again at the user-approval gate (Step 4).

On **(c) Clarify**:
- Capture the user's relationship description, decide one vs. multiple based on it, proceed accordingly.

### When to skip Step 1.5 entirely

- The user's prompt is a single clear capability (e.g. "add a /healthcheck endpoint"). No prompt needed.
- The user is invoking `/sdd:update` (this skill runs the same template but the multi-feature decision was already made when the original spec was written).
- The user explicitly said "one spec for all of this" in their initial message.

When in doubt, ask. The 30 seconds for an AskUserQuestion is much cheaper than the 30 minutes of re-iterating an oversized spec.

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

**Multi-service note:** if Step 0 chose option (a), insert a `Services:` line directly under the H1 title (between the title and Section 1), listing the services this feature targets:

```markdown
# Spec: <feature>

Services: backend, frontend

## 1. Goal
```

For multi-service specs, Section 3 (Acceptance Criteria) groups ACs under `### <ServiceName> (service: <name>)` subsection headers; single-service specs keep a flat AC list (no `### ...` subsections). See the Section 3 example below.

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

**Multi-service format** (only when the spec has a `Services:` line — see above): group AC clusters under per-service subsection headers so each AC inherits its service tag:

```markdown
## 3. Acceptance Criteria

### Backend (service: backend)

- **AC-1.1:** POST /api/login returns 200 with a JWT on valid credentials.
- **AC-1.2:** POST /api/login returns 401 with body `{"error":"invalid credentials"}` on bad password.

### Frontend (service: frontend)

- **AC-2.1:** LoginForm submits credentials and stores the returned JWT in localStorage.
- **AC-2.2:** On 401 response, LoginForm shows the inline error text "Invalid credentials".
```

`/sdd:build` reads each AC's enclosing subsection header to resolve its service, then uses that service's test command and layout profile from `CLAUDE.md`. ACs without a subsection header in a multi-service spec are an error — every AC must belong to exactly one service.

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

## Step 4 — Confirm + explicit user-approval gate

Print one short confirmation listing:

- The path written (`docs/specs/$1/spec.md`).
- The list of AC-IDs and VS-IDs you assigned.
- Any open questions still unresolved.

Then AskUserQuestion to gate the cycle:

> Spec written to `docs/specs/$1/spec.md`. Please review it now and decide:
> - **(a) Approve** — the spec captures what you want; `/sdd:build` can begin per-AC RGR
> - **(b) Edit** — I'll revise sections; tell me what to change
> - **(c) Cancel** — leave the file as a draft; don't mark the spec phase done

On **(a) Approve**:

1. Initialize or update `docs/specs/$1/spec-status.md` with a `## Phase progress` block at the top, immediately after the `Latest review:` line (or insert both if the file doesn't exist yet). The Phase 1 row is set to `done`:

   ```markdown
   # spec-status: <feature>

   Last updated: <YYYY-MM-DD>
   Latest review: (none yet)

   ## Phase progress

   | Phase           | Status      | Updated      | Notes                |
   |-----------------|-------------|--------------|----------------------|
   | 1. spec         | done        | <YYYY-MM-DD> | approved by user     |
   | 2. build        | pending     | —            | —                    |
   | 3. review       | pending     | —            | —                    |
   | 4. fix          | pending     | —            | —                    |
   | 5. validate     | pending     | —            | —                    |
   | 6. ship         | pending     | —            | —                    |

   ## Status per Acceptance Criterion

   | AC-ID | Status | Notes |
   |---|---|---|
   <one row per AC-ID, all status = not-started>
   ```

   If `spec-status.md` already exists from a prior cycle (`/sdd:update`), Edit the file:
   - Mark Phase 1 row Status = `done`, Updated = today, Notes = `"approved by user (re-approved <YYYY-MM-DD>)"`.
   - For added/modified ACs from this update, set their per-AC row to `not-started` / `stale` (already done by `/sdd:update`).

2. Output the next-step pointer:

   ```
   Spec approved for `$1`.
   Next: /sdd:build $1
     /sdd:build will detect the test framework + layout profile, scaffold tests,
     and iterate each AC via per-AC red-green-refactor (one failing test → minimal
     impl → green verified → next AC).
   ```

On **(b) Edit**: capture the user's requested changes inline, apply them with Edit, then loop back to Step 4 (re-confirm + re-prompt). Do NOT mark Phase 1 = `done` until the user picks Approve.

On **(c) Cancel**: leave `spec.md` on disk but DO NOT create or update `spec-status.md`. Output:

```
Spec draft saved to `docs/specs/$1/spec.md` but not approved.
Re-run /sdd:spec $1 when ready to approve and proceed.
```

Stop. `/sdd:build` will refuse to run on an unapproved spec — see `/sdd:build`'s pre-check 0.

DO NOT proceed to building in this skill — the user must explicitly trigger `/sdd:build`.

## Red Flags — STOP and reset

| Thought | Reality |
|---|---|
| "I'll skip Step 1.5 (scope check) for small-looking requests" | Small-looking requests with "and also" become oversized specs. The 30 seconds for an AskUserQuestion is much cheaper than re-iterating later. |
| "I'll auto-approve at Step 4 since the spec looks fine" | The user has to approve. The Phase 1 gate is for the human, not for you. Always present `(a)/(b)/(c)` and wait. |
| "I'll write `/sdd:tests $1` as the next step" | `/sdd:tests` is deprecated. Always point at `/sdd:build $1`. |
| "Error path AC without a discriminating signal is fine, the test phase will figure it out" | `/sdd:build`'s RED step refuses to write a too-permissive test. Spec the signal now, or fix it after the AC fails RED. |
| "The user gave vague requirements — I'll fill in details" | Ask. Step 2 exists for exactly this. Don't invent the spec; capture what the user actually wants. |

## Rules

- **Never implement code.** Only write the spec.
- **Never reuse an `AC-ID` or `VS-ID`** that already exists in the file (in update mode, append new IDs — don't renumber).
- **Never guess at stack or conventions** — ask in Step 2.
- **Keep it tight** — under 200 lines if possible. No padding, no repetition.
- **Every error path in Section 3 names a discriminating signal.** Vague error contracts ("an error is shown") produce false-pass tests.
- **Sub-capabilities of one feature = one spec, multiple AC-IDs.** Don't split.
