# SDD Init Notes — How to Build an `sdd:init` Skill

This document captures the exact steps performed to migrate LKMS from flat spec files into the SDD plugin format. Use it as the blueprint for a future `sdd:init` plugin skill that can do this automatically for any new project.

---

## 1. What problem `sdd:init` solves

A project may already have specs written as flat markdown files in `docs/specs/` before the SDD plugin is installed. The plugin's `sdd:build`, `sdd:validate`, `sdd:status`, and `sdd:ship` skills all expect specs in the subdirectory layout (`docs/specs/<feature>/spec.md` + `spec-status.md`). Without `sdd:init`, teams either skip using the plugin or have to manually restructure their docs.

`sdd:init` bridges the gap: it reads existing flat spec files, restructures them into SDD-compatible directories, adds `US-N` IDs to user stories, scaffolds `spec-status.md` and `docs/codebase-map.md`, and updates `CLAUDE.md` to reference the new paths.

---

## 2. What init scans for

**Input detection:**

- Looks for `docs/specs/*.md` flat files (files at the root of `docs/specs/`, not in subdirectories).
- For each file, reads the content to detect the presence of a `## 3. User Stories` section (or equivalent).
- Files **without** a User Stories section (e.g. `00-project-overview.md`) are classified as **project-level docs** and offered a separate move target (`docs/project-overview.md` or similar).
- Files **with** a User Stories section are classified as feature specs and queued for migration.
- Also reads `CLAUDE.md` (if present) to find an existing spec index table — used for cross-referencing feature names.

**Naming convention detection:**

- Filename pattern: `NN-slug.md` — the numeric prefix is stripped to derive the feature directory name.
- Example: `02-auth.md` → directory `auth/`, `03-user-and-skills.md` → directory `user-and-skills/`.
- If no numeric prefix exists, use the bare filename stem as the feature name.

---

## 3. How init creates the directory structure

For each detected feature spec file `NN-<slug>.md`:

1. Derive kebab-case feature name by stripping the numeric prefix and any leading zeros: `"02-auth.md"` → `"auth"`.
2. Check if `docs/specs/<feature>/` already exists — if yes, **skip** (idempotent; print a warning).
3. Create `docs/specs/<feature>/` directory.
4. Write `docs/specs/<feature>/spec.md` with the following transformations applied to the original content:
   - Prepend `# Spec: <feature-name>` as the first line, followed by a blank line.
   - Rename the title line (e.g. `# Spec 02 — Authentication`) to match the standard `# Spec: <feature>` format.
   - Ensure section headings match the SDD template (`## 1. Goal`, `## 2. Requirements`, `## 3. User Stories`, `## 4. Technical details`, `## 5. Out of scope`, `## 6. Edge cases / open questions`). Rename non-standard headings (e.g. `## 2. Tables` → `## 4. Technical details` if it contains implementation details).
   - In Section 3, assign US-N IDs to story bullets (see Section 4 below).
   - Scaffold empty `  - Done when:` sub-bullets under each story for the user to fill in.
5. For project-level docs, write to `docs/<slug>.md` (not inside `specs/`).

---

## 4. US-N ID assignment algorithm

**Global counter, not per-spec:**

- Start a counter at `US-1`.
- Process spec files in **filename sort order** (so `01-...` gets US-1, `02-...` gets the next ID after spec 01's stories, etc.).
- For each `- As a …` or `- As an …` bullet in Section 3, assign the current counter value and increment.
- Bullet becomes: `- **US-N**: As a …`

**Idempotency:**

- If a bullet already starts with `- **US-N**:` (any number), skip it — do not reassign.
- This allows running init a second time without corrupting existing IDs.

**Why global IDs:**

- The `sdd:status` skill scans all `spec-status.md` files and displays a project-wide story table. Globally unique IDs prevent ambiguity (e.g. two "US-1" from different specs).

**Edge case — no User Stories section:**

- If Section 3 is absent or is a placeholder like "All 9 stories from spec 00", prompt the user to either supply the stories interactively or skip ID assignment for that spec.
- For the `frontend` spec in LKMS, Section 3 said "All 9 in-scope user stories from the MVP cut" — the init process expanded this by reading the project overview and listing the 9 stories explicitly.

---

## 5. spec-status.md defaults

**Default behavior (new project, not yet implemented):**

- Create `docs/specs/<feature>/spec-status.md` with all stories set to `not-started`.
- Set `Last updated:` to today's date.
- Never overwrite an existing `spec-status.md` (it may contain live in-progress tracking).

**Already-implemented detection:**

- Use a `--all-done` flag to set all stories to `done` (used when migrating a finished project like LKMS).
- Alternatively: offer a heuristic — if `git log --oneline -- <feature-related-paths>` returns commits, prompt the user: "This feature looks implemented. Set all stories to `done`? [y/n]".

**Format:**

```markdown
# spec-status: <feature>

Last updated: YYYY-MM-DD

## Status per User Story

| US-ID | Status      | Notes |
| ----- | ----------- | ----- |
| US-N  | not-started |       |
```

Valid status values: `done`, `in-progress`, `blocked`, `not-started`.

---

## 6. codebase-map.md generation strategy

The init command should **scaffold, not describe** — it generates the table rows with placeholder role strings. The developer fills in the descriptions.

**Language detection:**

- `pom.xml` present → Java/Maven project.
- `package.json` present at root → Node.js project.
- Both present → monorepo (like LKMS): treat as Java + React.

**File discovery rules by language:**

For Java/Maven (find under `src/main/java/`):

- `*Application.java` → Spring Boot entry point
- `*Controller.java` → REST controller
- `*Service.java` → business logic
- `*Repository.java` → data access
- `*Entity.java` or entity classes in `entity/` packages
- `*Config.java` → configuration classes
- `V*.sql` under `db/migration/` → Flyway migrations

For React/Vite (find under `src/`):

- `main.jsx` / `main.tsx` → entry point
- `App.jsx` / `App.tsx` → router root
- `*Page.jsx` / `*Page.tsx` → page-level components
- `*Context.jsx` → React context providers
- `axiosInstance.js` / `apiClient.js` → HTTP client config
- `*Api.js` files in `api/` → API call modules

**Output format:**

```markdown
| `path/to/file.java` | <!-- TODO: describe role --> |
```

Instruct the user to replace `<!-- TODO: describe role -->` with a one-line description after init completes.

**What NOT to include:**

- Test files (test sources, `*.spec.js`, `*Test.java`)
- Build output directories (`target/`, `dist/`, `node_modules/`)
- Config files that are self-explanatory (`pom.xml`, `package.json`, `.gitignore`)

---

## 7. Manual steps after init runs

These steps cannot be automated and require human judgment:

1. **Fill in Done-when blocks**: For each story's `Done-when:` scaffold, write either:
   - An `*(automated)*` curl/shell command that tests the API endpoint.
   - A `*(manual)*` description of what to verify in the browser.
2. **Fill in codebase-map.md role descriptions**: Replace each `<!-- TODO: describe role -->` with a one-line description of what the file does.
3. **Update CLAUDE.md spec index**: Change the `## Spec index` table to reference the new subdirectory paths.
4. **Delete old flat files**: After verifying the new subdirectory specs look correct, remove the original `docs/specs/NN-*.md` flat files.
5. **Move project-level docs**: Move the project overview file to `docs/project-overview.md` (or wherever it was suggested).
6. **Verify with `/sdd:status`**: Run `/sdd:status` to confirm all features are visible and story counts are correct.

---

## 8. Idempotency rules

The init command must be safe to run multiple times:

- **Existing subdirectory spec detected:** Print `"Skipping auth — docs/specs/auth/spec.md already exists"`. Do not overwrite.
- **Existing spec-status.md detected:** Print `"Skipping auth spec-status.md — already exists"`. Never overwrite (it may contain live tracking data).
- **Existing codebase-map.md detected:** Offer to append newly discovered files, not replace the whole file.
- **US-N IDs already present in spec.md:** Do not re-assign. Any bullet already formatted `**US-N**:` is left untouched.

---

## 9. What was done for LKMS (reference migration)

For reference, here is exactly what was done to migrate LKMS:

| Step | Action                                                                                                                     |
| ---- | -------------------------------------------------------------------------------------------------------------------------- |
| 1    | Read all 9 flat spec files in parallel                                                                                     |
| 2    | Created `docs/specs/<feature>/spec.md` for 8 features (01–08), applying: `# Spec:` header, US-N IDs, Done-when blocks      |
| 3    | Moved project overview content to `docs/project-overview.md` (not a trackable feature spec)                                |
| 4    | Created `docs/specs/<feature>/spec-status.md` for all 8 features with all stories = `done` (project was fully implemented) |
| 5    | Created `docs/codebase-map.md` with ~50 key files mapped                                                                   |
| 6    | Updated `CLAUDE.md` spec index to reference new subdirectory paths                                                         |
| 7    | Deleted the 9 original flat files                                                                                          |

**US-ID range summary:**

| Feature          | US-IDs        |
| ---------------- | ------------- |
| data-model       | US-1          |
| auth             | US-2 – US-3   |
| user-and-skills  | US-4 – US-7   |
| course           | US-8 – US-11  |
| enrollment       | US-12 – US-14 |
| training-request | US-15 – US-19 |
| admin-dashboard  | US-20 – US-22 |
| frontend         | US-23 – US-31 |

Total: 31 user stories across 8 feature specs.
