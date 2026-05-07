# review-code — Two-Pass Code Review for Claude Code

A Claude Code plugin that runs a quality reviewer and a security reviewer **in parallel**, then aggregates both into a single timestamped Markdown report on disk. One slash command, three subagents, read-only by design.

## The flow

```
/review-code:review-code <path>
       │
       ├── code-quality-reviewer   →  # Code Quality Review  (Markdown)
       ├── security-reviewer       →  # Security Review      (Markdown)
       │   (run in parallel)
       │
       └── code-reporter           →  ./reports/code-review_<timestamp>.md
```

The two reviewers never overlap — quality findings stay on the maintainability side, security findings stay on the security side. If one reviewer spots an issue in the other's domain, it notes it once with `(out of scope, see <other-reviewer>)` and moves on. The reporter aggregates both reports verbatim and never re-judges, re-ranks, or invents findings.

## Slash command

```
/review-code:review-code <path>
```

`<path>` can be a file, a directory, a feature folder, or a diff range — anything the reviewers can scope to. When invoked, the command:

1. Dispatches `code-quality-reviewer` and `security-reviewer` against the target in parallel.
2. Once both finish, dispatches `code-reporter` with the two outputs.
3. The reporter writes `./reports/code-review_<YYYY-MM-DD_HHMMSS>.md` and replies with the absolute path + a 2–3 sentence summary.

The reporter never overwrites an existing file; on collision it appends `_a`, `_b`, ..., then a 4-character hex suffix.

## The three agents

### `code-quality-reviewer` (maintainability)

SonarQube-style review across 10 dimensions: cyclomatic complexity, function/file length, duplication, naming clarity, dead code, magic numbers, deep nesting, error handling, inconsistent patterns, and test coverage gaps. Language-aware idiom checks for Python, JavaScript/TypeScript, Java, Go, and Rust. Severity rubric: Blocker / Critical / Major / Minor / Info.

### `security-reviewer` (OWASP / CWE)

OWASP Top 10 (2021)-aligned review across application security (injection, XSS, CSRF, SSRF, XXE, path traversal, deserialization, open redirects), authentication / authorization (broken access control, IDOR, session management, JWT misuse, password storage), cryptography (weak algorithms, hardcoded keys, disabled cert verification), secrets / sensitive data, configuration / dependencies (security headers, CORS, vulnerable libs), and input handling (ReDoS, file uploads, mass assignment). Each finding maps to a CWE and an OWASP category. Critical / High findings include a one-line CVSS-style attack vector + impact note. Severity rubric: Critical / High / Medium / Low / Informational.

### `code-reporter` (aggregator)

Writes the combined report. Performs **no code analysis of its own** — purely faithful aggregation. Builds an Executive Summary, a Combined Severity Dashboard mapping quality → security severities (Blocker→Critical, Major→Medium, etc.), and a Top Priorities list across both reviewers. Embeds each reviewer's full report verbatim under its section. Adds metadata (timestamp, scope, git short SHA, model names, total findings, absolute output path).

## Optional scanner integrations

Each reviewer looks for and runs standard scanners if they're available, **always in check-only mode**. Missing tools are logged in the `Scanners run:` line and skipped — never an error.

| Concern         | Tools tried                                                                                            |
| --------------- | ------------------------------------------------------------------------------------------------------ |
| Lint / types    | `eslint`, `tsc --noEmit`, `ruff`, `pylint`, `mypy`, `golangci-lint`, `go vet`, `mvn spotless`, `rubocop` |
| Secrets         | `gitleaks`, `trufflehog`                                                                               |
| SAST            | `semgrep` (OWASP Top 10 + auto), `bandit`, `gosec`                                                     |
| Dependency CVEs | `npm audit`, `pip-audit`, `safety`, `pnpm audit`, `govulncheck`, `mvn dependency-check`                |
| Containers      | `trivy fs`, `trivy config`                                                                             |

Scanner output is treated as a starting point. Each reviewer re-reads flagged code in context, drops false positives, and promotes underrated findings.

## Read-only guarantee

All three agents have only read-and-report tooling — `Read`, `Grep`, `Glob`, `Bash`, plus `Write` for the reporter (the only file it writes is the timestamped report under `./reports/`). The plugin will never edit, refactor, or "fix up" your code. If you want fixes, the report tells you exactly what to change and where; you (or another tool) make the edit.

## Output

```
./reports/code-review_<YYYY-MM-DD_HHMMSS>.md
```

The report has these sections, in order:

- Executive Summary (3–5 sentences for a non-technical lead)
- Combined Severity Dashboard
- Top Priorities (5–7 items consolidated across both reviewers)
- Security Findings (verbatim `# Security Review` block)
- Code Quality Findings (verbatim `# Code Quality Review` block)
- Metadata

The reporter prints **only** the absolute path + a brief summary back to chat — the full report stays on disk. Open it with your editor or pipe it through `glow`, `mdcat`, etc.

## Dependencies

- `git` — used by the reporter only, to capture the short SHA in the metadata block. If absent or the directory isn't a repo, the SHA line says `not available` and the report is still written.
- The optional scanner tools listed above. None are required.

The plugin itself is language-agnostic and depends on no other Claude Code plugins.
