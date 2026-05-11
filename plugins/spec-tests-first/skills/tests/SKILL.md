---
name: tests
description: DEPRECATED in STF v2 — folded into /sdd:build. Use when the user invokes /sdd:tests <feature>; this skill no longer generates tests. It prints a one-line redirect telling the user to run /sdd:build directly (which now handles test scaffolding + per-AC red-green-refactor test writing in one phase) and stops without writing any files.
argument-hint: <feature-name>
allowed-tools: Read
---

# /sdd:tests — DEPRECATED (folded into /sdd:build)

This skill is a deprecation shim. In STF v2 the dedicated tests phase no longer exists — test scaffolding and per-AC test writing are now both handled by `/sdd:build` via the per-AC red-green-refactor loop.

Output to the user:

```
/sdd:tests is deprecated in STF v2. Test scaffolding and per-AC test writing
are now handled by /sdd:build via the per-AC red-green-refactor loop.

Run /sdd:build $1 directly. It will:
  1. Resolve the test framework + layout profile from CLAUDE.md (or detect + ask).
  2. Scaffold the test directory (or no-op for co-located profiles like Angular/Go).
  3. Iterate each AC: write a failing test, watch it fail via test-runner,
     write minimal implementation, watch it pass, regression-check the feature suite.
  4. Commit the whole feature once at the end.
```

Then stop. Do not invoke any other skill. Do not write any files. Do not propose any further action.
