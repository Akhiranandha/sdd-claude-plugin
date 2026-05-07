---
name: validate
description: Phase 4 of the SDD cycle. Use when the user invokes /sdd:validate <feature> to walk through the spec's "Validation steps" section after implementation. Categorizes steps as automated (executes them) or manual (presents as a checklist), aggregates pass/fail, and either unblocks the ship phase or loops back to build with failure notes appended to spec-status.md.
argument-hint: <feature-name>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion
---

# /sdd:validate — Phase 4: Validate Against the Spec

You are running Phase 4 of the SDD cycle for feature **$1**.

This phase is the **acceptance gate**. Tests passing means the code is *correct*; validation passing means the *feature works*. They are not the same.

## Steps

1. **Read the spec.** Open `docs/specs/$1/spec.md`. Extract the **Validation steps** section. If it is empty or missing, stop and tell the user:

   > The spec has no validation steps. Add some to `docs/specs/$1/spec.md` (concrete steps to verify the feature works, each with a `VS-N` ID), then re-run `/sdd:validate $1`.

2. **Parse each step's `VS-N` ID.** Every step in the spec must carry a stable ID (e.g. `VS-1`, `VS-2`). If any step lacks an ID, stop and ask the user to add IDs first — failure logs reference these IDs, and renumbering on the fly makes failure history unreadable.

3. **Categorize each step** as one of:
   - **Automated** — anything you can run as a shell command, script, HTTP call, or test invocation.
   - **Manual** — anything that requires a human (UI clicks, visual checks, "log in and verify the dashboard renders").

4. **Run automated steps.** Execute each via Bash. Capture stdout/stderr and exit code. Mark pass/fail per `VS-ID`. Show the result of each to the user inline (`VS-1: PASS`, `VS-2: FAIL — expected X, got Y`).

5. **Walk manual steps.** For each manual step, present it clearly (prefixed with its `VS-ID`) and use AskUserQuestion to ask:

   > **VS-N:** [step text]. Did this step pass? **(y / n / skip)**

   Record the answer. `skip` counts as neither pass nor fail; surface it in the summary.

6. **Aggregate.**
   - **All pass** → output:

     > Validation complete — all checks pass.
     > Run `/sdd:ship $1` to review, commit, and ship.

   - **Any fail** → append a `## Validation failures (YYYY-MM-DD)` section to `docs/specs/$1/spec-status.md` listing each failed `VS-ID` and the observed behaviour. Then output:

     > Validation failed: [list of VS-IDs]. Details appended to `docs/specs/$1/spec-status.md`.
     > Fix and re-run `/sdd:build $1`. If requirements changed, run `/sdd:update $1` first.
