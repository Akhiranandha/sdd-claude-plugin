---

name: validate
description: Phase 3 of the SDD cycle. Use when the user invokes /spec-flow:validate <feature> to walk through each User Story's Done-when checks after implementation. Categorizes checks as automated (executes them) or manual (presents as a checklist), falls back to a manual y/n confirmation for stories with no Done-when block, aggregates done/blocked per story, and either unblocks the ship phase or loops back to build with failure notes appended to spec-status.md.
argument-hint: <feature-name>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion

---

# /spec-flow:validate — Phase 3: Validate Against the Spec

You are running Phase 3 of the SDD cycle for feature **$1**.

This phase is the **only** correctness gate. Code compiling and self-review passing means the implementation looks plausible; validation passing means each User Story actually works end-to-end.

## Steps

1. **Read the spec.** Open `docs/specs/$1/spec.md`. Extract every story from Section 3 — both the narrative line and any `Done when:` sub-block beneath it. If Section 3 is empty or missing, stop and tell the user:

   > The spec has no User Stories. Add some to `docs/specs/$1/spec.md` (each with a `US-N` ID), then re-run `/spec-flow:validate $1`.

2. **Parse each story's `US-N` ID.** Every story in the spec must carry a stable ID (e.g. `US-1`, `US-2`). If any story lacks an ID, stop and ask the user to add IDs first — failure logs reference these IDs, and renumbering on the fly makes failure history unreadable.

3. **For each story, walk its checks.** There are three cases:

   ### Case A — Story has Done-when checks

   Categorize each check inside the block as one of:
   - **Automated** — anything you can run as a shell command, script, or HTTP call.
   - **Manual** — anything that requires a human (UI clicks, visual checks, "log in and verify the dashboard renders").

   - **Run automated checks** via Bash. Capture stdout/stderr and exit code. Report inline (`US-1 [auto check 1]: PASS`, `US-1 [auto check 2]: FAIL — expected X, got Y`).
   - **Walk manual checks** by presenting each clearly (prefixed with the `US-N` ID) and using AskUserQuestion:

     > **US-N (manual):** [check text]. Did this check pass? **(y / n / skip)**

     Record the answer. `skip` counts as neither pass nor fail; surface it in the summary.

   ### Case B — Story has no Done-when block

   Present the story narrative to the user via AskUserQuestion:

   > **US-N**: [story text]. This story has no Done-when checks. Manually confirm it works as described? **(y / n / skip)**

   ### Case C — Story is marked `not-started` in spec-status.md

   Skip it. Surface in the summary as "skipped — not implemented yet."

   Note: `blocked` stories are NOT skipped — they're walked through Case A or B as normal. The user may have fixed the underlying issue between runs; re-validating confirms it.

4. **Aggregate per story.** A story is `done` iff every check (auto + manual) for it is `y` / PASS. If any check is `n` / FAIL, the story is `blocked`. If a story is mixed (some pass, some skip, no fail), report it as `in-progress` and keep its prior status unchanged.

5. **Report and update spec-status.md.**
   - **All stories `done`** → update each story's row in `docs/specs/$1/spec-status.md` to `done`, then output:

     > Validation complete — all stories pass.
     > Run `/spec-flow:ship $1` to review, commit, and ship.

   - **Any `blocked`** → append a `## Validation failures (YYYY-MM-DD)` section to `docs/specs/$1/spec-status.md` listing each blocked `US-N`, the failing check (or "manual: n"), and the observed behaviour. Update the row's status to `blocked`. Then output:

     > Validation failed: [list of US-IDs]. Details appended to `docs/specs/$1/spec-status.md`.
     > Fix and re-run `/spec-flow:build $1`. If requirements changed, run `/spec-flow:update $1` first.
