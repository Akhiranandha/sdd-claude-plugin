---
name: tests
description: DEPRECATED in STF v2 — folded into /spec-tests-first:build. Use when the user invokes /spec-tests-first:tests <feature>; this skill no longer generates tests. It prints a one-line redirect telling the user to run /spec-tests-first:build directly (which now handles test scaffolding + per-AC red-green-refactor test writing in one phase) and stops without writing any files.
argument-hint: <feature-name>
allowed-tools: Read
---

# /spec-tests-first:tests — DEPRECATED (folded into /spec-tests-first:build)

**Announce at start:** Say to the user: "/spec-tests-first:tests is deprecated in STF v2. I'll print the redirect and stop — no files will be touched." Then proceed.

This skill is a deprecation shim. In STF v2 the dedicated tests phase no longer exists — test scaffolding and per-AC test writing are now both handled by `/spec-tests-first:build` via the per-AC red-green-refactor loop.

## Iron Law

> **This skill writes nothing. Prints the redirect message and stops. Never falls back to "old behavior" — there is no fallback.**

Output to the user:

```
/spec-tests-first:tests is deprecated in STF v2. Test scaffolding and per-AC test writing
are now handled by /spec-tests-first:build via the per-AC red-green-refactor loop.

Run /spec-tests-first:build $1 directly. It will:
  1. Resolve the test framework + layout profile from CLAUDE.md (or detect + ask).
  2. Scaffold the test directory (or no-op for co-located profiles like Angular/Go).
  3. Iterate each AC: write a failing test, watch it fail via test-runner,
     write minimal implementation, watch it pass, regression-check the feature suite.
  4. Commit the whole feature once at the end.
```

Then stop. Do not invoke any other skill. Do not write any files. Do not propose any further action.
