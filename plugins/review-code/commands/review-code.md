---
name: review-code
description: Run a full two-pass code review on a path — code-quality-reviewer (maintainability) and security-reviewer (OWASP/CWE) in parallel, then code-reporter aggregates the results into ./reports/code-review_<timestamp>.md.
argument-hint: <path>
allowed-tools: Task, Read, Glob, Bash
---

Run a full code review on $ARGUMENTS:

1. In parallel, invoke:
   - code-quality-reviewer on the target path
   - security-reviewer on the target path
2. Once both complete, invoke code-reporter with both results
3. Save the report to reports/ with a timestamp
