# Codebase-map seeding patterns

Per-profile glob patterns + role templates for `/sdd:init`'s codebase-map scaffolding step. The `init` skill reads the section matching the resolved profile from `CLAUDE.md ## Test layout` and applies the globs to discover files. Role descriptions are left as `<!-- TODO: describe role -->` placeholders for the developer to fill in.

Each section has three blocks:

- **Include** — glob patterns to add to the map.
- **Exclude** — patterns to skip even if they match Include (tests, build output, generated code).
- **Role templates** — optional pattern-specific role hints that beat the generic TODO.

---

## profile: python-pytest

### Include
- `**/*.py` (excluding tests, virtualenvs, build output)

### Exclude
- `tests/**`
- `test_*.py` (anywhere)
- `*_test.py`
- `.venv/**` / `venv/**`
- `build/**` / `dist/**`
- `__pycache__/**`
- `setup.py`, `conftest.py` (fixture infrastructure, not source)

### Role templates
- `**/main.py` → "entry point"
- `**/cli.py` → "command-line interface"
- `**/api/*.py` → "API module"
- `**/models/*.py` → "data model"
- `**/services/*.py` → "service layer"

---

## profile: python-unittest

Same as `python-pytest`. Test naming differs (TestCase classes vs pytest functions), but source files are the same.

---

## profile: js-jest

### Include
- `src/**/*.{js,jsx,ts,tsx}` (excluding tests)
- `lib/**/*.{js,ts}`

### Exclude
- `**/__tests__/**`
- `**/*.test.{js,jsx,ts,tsx}`
- `**/*.spec.{js,jsx,ts,tsx}`
- `node_modules/**`
- `dist/**` / `build/**`
- `*.config.{js,ts}` (build config, not source)

### Role templates
- `src/index.{js,ts}` → "entry point"
- `src/server.{js,ts}` → "server entry"
- `src/routes/**.{js,ts}` → "route handler"
- `src/middleware/**.{js,ts}` → "Express middleware"
- `src/models/**.{js,ts}` → "data model"
- `src/services/**.{js,ts}` → "service layer"

---

## profile: js-vitest

Same as `js-jest` — Vitest discovers the same file patterns.

---

## profile: react-jest-rtl

### Include
- `src/**/*.{jsx,tsx}` (components)
- `src/**/*.{js,ts}` (hooks, utils, context)
- `src/api/**` (API client)

### Exclude
- `**/__tests__/**`
- `**/*.test.{jsx,tsx}`
- `**/*.spec.{jsx,tsx}`
- `node_modules/**`
- `build/**` / `dist/**`
- `src/setupTests.{js,ts}` (test infrastructure)

### Role templates
- `src/main.{jsx,tsx}` → "Vite/CRA entry point"
- `src/App.{jsx,tsx}` → "router root"
- `src/pages/**Page.{jsx,tsx}` → "page component"
- `src/components/**.{jsx,tsx}` → "presentational component"
- `src/hooks/use*.{js,ts}` → "custom React hook"
- `src/context/**Context.{jsx,tsx}` → "React context provider"
- `src/api/**.{js,ts}` → "API client module"
- `src/store/**.{js,ts}` → "state management"

---

## profile: angular-jasmine

### Include
- `src/app/**/*.ts` (components, services, modules)
- `src/environments/*.ts`

### Exclude
- `**/*.spec.ts` (Jasmine tests, co-located)
- `node_modules/**`
- `dist/**`
- `*.config.ts` / `karma.conf.js`

### Role templates
- `src/main.ts` → "Angular bootstrap"
- `src/app/app.module.ts` → "root NgModule"
- `src/app/**/*.component.ts` → "Angular component"
- `src/app/**/*.service.ts` → "Angular service"
- `src/app/**/*.module.ts` → "feature NgModule"
- `src/app/**/*.guard.ts` → "route guard"
- `src/app/**/*.pipe.ts` → "Angular pipe"

---

## profile: spring-boot-junit5

### Include
- `src/main/java/**/*.java`
- `src/main/resources/db/migration/V*.sql` (Flyway migrations)
- `src/main/resources/application*.yml`
- `src/main/resources/application*.properties`

### Exclude
- `src/test/**`
- `target/**`
- `*.class`

### Role templates
- `**/*Application.java` → "Spring Boot entry point"
- `**/*Controller.java` → "REST controller"
- `**/*Service.java` → "business logic"
- `**/*ServiceImpl.java` → "service implementation"
- `**/*Repository.java` → "JPA repository"
- `**/entity/*.java` or `**/*Entity.java` → "JPA entity"
- `**/dto/*.java` → "data transfer object"
- `**/*Config.java` → "Spring configuration"
- `**/*Filter.java` → "servlet filter"
- `**/V*.sql` → "Flyway migration"
- `**/application*.yml` → "Spring profile config"

---

## profile: go

### Include
- `**/*.go` (excluding tests)
- `cmd/**/main.go`
- `internal/**/*.go`
- `pkg/**/*.go`

### Exclude
- `**/*_test.go` (co-located tests)
- `vendor/**` (if present)

### Role templates
- `cmd/**/main.go` → "binary entry point"
- `internal/**/*_handler.go` → "HTTP handler"
- `internal/**/*_service.go` → "service layer"
- `internal/**/*_repository.go` → "repository / DB access"
- `pkg/**` → "public package"

---

## profile: rust

### Include
- `src/**/*.rs`

### Exclude
- `target/**`
- `tests/**` (integration tests — separate)
- `benches/**`
- `examples/**`

### Role templates
- `src/main.rs` → "binary entry point"
- `src/lib.rs` → "library root"
- `src/bin/*.rs` → "additional binary"

---

## profile: dotnet-xunit

### Include
- `**/*.cs` in projects NOT ending in `.Tests`

### Exclude
- `**/*.Tests/**` (test projects)
- `**/bin/**`
- `**/obj/**`

### Role templates
- `**/Program.cs` → "host entry point"
- `**/Startup.cs` → "ASP.NET startup configuration"
- `**/Controllers/*.cs` → "ASP.NET Core controller"
- `**/Services/*.cs` → "service layer"
- `**/Models/*.cs` → "data model"

---

## profile: ruby-rspec

### Include
- `app/**/*.rb`
- `lib/**/*.rb`
- `config/routes.rb`
- `db/migrate/*.rb`

### Exclude
- `spec/**`
- `tmp/**`
- `vendor/**`
- `*.gemspec`

### Role templates
- `app/controllers/**.rb` → "Rails controller"
- `app/models/**.rb` → "ActiveRecord model"
- `app/services/**.rb` → "service object"
- `app/jobs/**.rb` → "background job"
- `config/routes.rb` → "Rails routes"
- `db/migrate/*.rb` → "Rails migration"

---

## profile: phpunit

### Include
- `src/**/*.php`
- `app/**/*.php`

### Exclude
- `tests/**`
- `vendor/**`
- `*Test.php` (anywhere)

### Role templates
- `src/index.php` / `public/index.php` → "entry point"
- `src/**Controller.php` → "controller"
- `src/**Service.php` → "service"
- `src/**Repository.php` → "repository"

---

## profile: custom

### Include
- Use whatever the user specified in CLAUDE.md `## Test layout` (custom `tests_root` is informational here; init uses the project's source paths separately).
- Fall back to: `src/**/*` (excluding tests by best-effort heuristics).

### Exclude
- `tests/**`, `__tests__/**`, `**/*.test.*`, `**/*.spec.*` (universal test patterns)
- `node_modules/**`, `vendor/**`, `target/**`, `build/**`, `dist/**` (universal build outputs)

### Role templates
- (none) — all matches default to `<!-- TODO: describe role -->`.

---

## Applying these patterns

For the resolved profile (or each service's profile in monorepos):

1. Run Glob with each Include pattern → collect matches.
2. Run Glob with each Exclude pattern → subtract matches.
3. For each remaining file:
   - Try each Role template's pattern; on first match, use that role string with `<noun>` filled in if possible (or `<!-- TODO: describe role -->` otherwise).
   - If no template matches, use `<!-- TODO: describe role -->`.
4. Emit a table row: `| path/to/file | <role or TODO> |`.

Group rows under `## <Service>` headers when multi-service; one flat `## Source files` block when single-service.
