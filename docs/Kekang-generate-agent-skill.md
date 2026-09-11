---
name: kekang-generate-agents
description: Generate a harness-engineering-compliant AGENTS.md tailored to the project's stack. Uses Graphify graph data when available for grounded architecture reflection. Supports Laravel, NestJS, Next.js (incl. static-export/Tauri), TypeScript, PHP, Python, Rust/Tauri, and combinations. Invoke via /kekang:generate-agents or automatically when user asks to "generate agents.md" or "create AGENTS.md for this project".
tools: Read, Write, Bash, Grep, Glob
---

# Kekang: Generate AGENTS.md Skill

Generate a high-quality, harness-engineering-compliant `AGENTS.md` at the project root. This skill embeds OpenAI + Anthropic harness engineering discipline into the generated file so that agents working on the project automatically follow verification, escalation, and reviewer protocols.

## When to Invoke

- User runs `/kekang:generate-agents`
- User asks to "generate an AGENTS.md" or "create AGENTS.md for this project"
- Kekang app spawns a `claude` CLI session with this skill via `ClaudeCliService` (request file at `.kekang/generate-request.md`)
- User wants to bootstrap or improve existing AGENTS.md

## Workflow (Follow in Order)

### Step 1: Gather Context

Check for context inputs in this order:

**1a. Request file from Kekang (if exists):**
```
Read: .kekang/generate-request.md
```

This file, when present, contains structured input:
- `stack:` — comma-separated stacks (e.g., "laravel,livewire" or "nestjs,typescript")
- `context:` — user's prose description of the project
- `graphify_report:` — path to Graphify's GRAPH_REPORT.md (usually `graphify-out/GRAPH_REPORT.md`)
- `existing_agents:` — path to existing AGENTS.md (if in improvement mode)
- `mode:` — `create` or `improve`

If this file doesn't exist, gather context interactively (Step 1b).

**1b. Interactive context gathering (fallback):**

If invoked directly (not via Kekang), do project discovery:

1. Detect stack via presence of marker files:
   - `composer.json` + `artisan` → Laravel
   - `composer.json` (no artisan) → PHP (framework unknown, ask user)
   - `package.json` with `@nestjs/core` in deps → NestJS
   - `package.json` with `next` in deps → Next.js
   - `package.json` with `typescript` but no framework → TypeScript
   - `requirements.txt` or `pyproject.toml` → Python (check for FastAPI, Django, Flask)
   - `Cargo.toml` + `src-tauri/` or `tauri.conf.json` → Rust/Tauri desktop app (check frontend: Next.js/Vite/Svelte)
   - `Cargo.toml` (no Tauri) → Rust (check for axum/actix/clap)
   - Mixed: report all detected stacks
2. Check for existing `AGENTS.md` — if exists, offer improvement mode
3. Check for `graphify-out/GRAPH_REPORT.md` — if exists, read it for grounded architecture data
4. Ask user for project description in one sentence:
   - "What does this project do at a high level? (e.g., 'admin panel for restaurant POS with Filament', 'REST API for e-commerce inventory')"
5. Ask about specific conventions/patterns:
   - "Any specific patterns or libraries this project uses that agents should know? (e.g., 'Actions pattern, Pest for tests, Livewire for reactive UI')"

### Step 2: Read Graphify Data (if available)

If Graphify report exists, this is the **most valuable context**. Read it and extract:

- **Actual architecture layers** (not what user thinks, what actually is)
- **Node relationships** (module dependencies)
- **God nodes** (files with too many connections — mention as tech debt)
- **Language distribution**
- **Entry points**
- **External dependencies**

**Critical:** If Graphify data contradicts user's description, trust Graphify. Note the discrepancy in AGENTS.md under "Reality Check" section.

If no Graphify report, do minimal code scan:
- `Glob` main directories (`src/`, `app/`, `lib/`, etc.)
- `Read` key config files (`package.json`, `composer.json`, `pyproject.toml`, `tsconfig.json`)
- Identify architectural pattern from directory structure

### Step 3: Select Template

Based on detected stack(s), select from templates below. For multi-stack projects, combine relevant sections.

### Step 4: Generate AGENTS.md

Follow the template structure. Rules:

- **Maximum 120 lines** (target 80-100). If longer, move detail to `docs/`.
- **Map/index style**, not encyclopedia. Reference other docs rather than duplicating.
- **Include harness sections** (Verification, Escalation, Reviewer, GC Compliance) — these are NON-NEGOTIABLE.
- **Concrete, not abstract**. Instead of "follow best practices", write "use Repository pattern, see `app/Repositories/UserRepository.php` for reference".
- **Ground claims in Graphify data** when available. Instead of "the app has 3 layers", write "the app has 3 layers (verified from graph: controllers, services, repositories with 47 modules)".

### Step 5: Write and Confirm

1. Write to `AGENTS.md` at project root (or update if in improvement mode)
2. Print summary to user:
   - Sections included
   - Stack detected
   - Whether Graphify was used
   - Any warnings (drift detected, missing patterns, etc.)
3. If Kekang triggered this via a managed `claude` session (ClaudeCliService), also write to `.kekang/agents-generation-log.md`:
   - Timestamp
   - Input context used
   - Output file path
   - Any decisions made

### Step 6: Cleanup

- Delete `.kekang/generate-request.md` (temporary file, consumed)
- Do NOT delete anything else

---

## Template: Laravel

Use when stack includes Laravel. Adapt sections marked `[project-specific]` to actual context.

```markdown
# AGENTS.md

## What This Is
[project-specific: 1-2 sentences describing the product/system]

## Stack
- Laravel [version detected from composer.json]
- PHP [version from composer.json require]
- [Filament/Livewire/Inertia/API-only — from composer.json deps]
- Testing: [Pest/PHPUnit — detect from composer.json]
- Static analysis: [PHPStan/Larastan — detect from composer.json]
- Database: [PostgreSQL/MySQL/SQLite — detect from .env.example or config]

## Architecture Layers
[If Graphify available, use actual detected layers. Otherwise use Laravel conventions:]
- `app/Http/Controllers/` — HTTP handlers only, no business logic
- `app/Services/` — business logic
- `app/Repositories/` — data access (if repository pattern used)
- `app/Models/` — Eloquent models + scopes + relationships
- `app/Actions/` — single-purpose operations (if Actions pattern used)
- `app/Filament/` — admin panel resources (if Filament used)
- `app/Livewire/` — reactive components (if Livewire used)
- `app/Jobs/` — queued jobs
- `app/Events/`, `app/Listeners/` — event system

## Golden Principles
- Never business logic in Controllers or Livewire components → route to Service
- All external API calls via dedicated Services, never in Models
- Use FormRequest for validation, never inline validation in Controllers
- Eloquent scopes for common queries, not raw DB::table in Controllers
- Actions for complex multi-step operations
- Tests wajib for all Services and Actions
- [project-specific: additional conventions from context]

## Verification Requirements (MANDATORY)
Before marking any task complete, agent MUST run:
1. `./vendor/bin/pest` [or phpunit] — all tests must pass
2. `./vendor/bin/pint --test` — code style must comply
3. `./vendor/bin/phpstan analyse` — static analysis must pass at project level
4. `php artisan test --parallel` — full suite for larger changes

If ANY of these fail after 2 fix attempts, ESCALATE — do NOT proceed.

## Escalation Protocol
When stuck after 2 attempts:
1. Write escalation to `.agent-workspace/escalations/[task-id].md`
2. Include: what failed, exact error, what you tried, what you suspect
3. STOP — do not proceed to next task
4. Human intervention required before continuing

## Reviewer Requirements
Two-stage review before merge:
1. **Spec-compliance reviewer:** Does implementation match the approved spec?
2. **Code-quality reviewer:** Standards, edge cases, security, performance

Both must pass. Reviewers are separate concerns — don't combine.

## GC Compliance
If you change architecture (add layer, new pattern, refactor):
- Update THIS AGENTS.md in the SAME PR
- Update relevant `docs/architecture/` files
- Update `docs/golden-principles/` if you introduce new principle
- Never let this file drift from actual implementation

Weekly GC scan will detect drift and create issues.

## Where Things Live
- Business decisions: `docs/decisions/`
- API contracts: `docs/api/`
- Runbooks: `docs/runbooks/`
- Migrations: `database/migrations/`
- Seeders: `database/seeders/`
- Feature tests: `tests/Feature/`
- Unit tests: `tests/Unit/`

## What NOT to Do
- Never `dd()` or `dump()` in committed code
- Never `->get()->count()` — use `->count()` for performance
- Never N+1 queries — always eager load (`->with()`)
- Never expose Eloquent models directly in API responses — use Resources
- Never bypass FormRequest validation
- Never delete migrations after they've been merged to main
- [project-specific: additional anti-patterns]

## Graphify Reality Check
[If Graphify was used and detected discrepancies with user's description, list them here]
[Otherwise omit this section]
```

---

## Template: NestJS

```markdown
# AGENTS.md

## What This Is
[project-specific description]

## Stack
- NestJS [version]
- TypeScript [version]
- Node.js [version from package.json engines]
- Testing: [Jest/Vitest]
- ORM: [TypeORM/Prisma/Drizzle — detect from deps]
- Validation: [class-validator/Zod — detect from deps]
- API style: [REST/GraphQL/tRPC — detect from deps]

## Architecture Layers
- `src/modules/` — feature modules (one per domain)
- `src/*/controllers/` — HTTP handlers
- `src/*/services/` — business logic
- `src/*/repositories/` — data access
- `src/*/dto/` — request/response DTOs with class-validator
- `src/*/entities/` — ORM entities
- `src/common/` — shared filters, interceptors, guards
- `src/config/` — configuration modules

## Golden Principles
- Never business logic in Controllers → delegate to Service
- Services never know about HTTP concerns (Request/Response)
- All input validation via DTOs with class-validator decorators
- Dependency injection always via constructor
- Repositories abstract database — never inject Repository directly to Controller
- One module = one bounded context
- [project-specific conventions]

## Verification Requirements (MANDATORY)
1. `npm run lint` — must pass
2. `npm run typecheck` or `tsc --noEmit` — must pass with strict mode
3. `npm test` — all tests pass
4. `npm run test:e2e` for API changes

If any fail after 2 attempts, ESCALATE.

## Escalation Protocol
Same as Laravel template above. Write to `.agent-workspace/escalations/[task-id].md`.

## Reviewer Requirements
Two-stage: spec-compliance THEN code-quality. Both required.

## GC Compliance
Architecture changes update this file + `docs/architecture/`.

## Where Things Live
- Decisions: `docs/decisions/`
- API contracts: `docs/api/` (or OpenAPI generated)
- Migrations: `src/database/migrations/` [adjust path]
- Test fixtures: `test/fixtures/`

## What NOT to Do
- Never `console.log` in committed code — use Logger service
- Never `any` type — use `unknown` and narrow
- Never inject Repository directly to Controller (bypasses Service layer)
- Never mutate DTO after validation
- Never expose entities directly in API — use DTO transformer
- Never skip decorators (@Injectable, @Controller, etc.)
- [project-specific]

## Graphify Reality Check
[If applicable]
```

---

## Template: Next.js

```markdown
# AGENTS.md

## What This Is
[project-specific]

## Stack
- Next.js [version] ([App Router/Pages Router — detect])
- React [version]
- TypeScript [version]
- Styling: [Tailwind/CSS Modules/styled-components]
- State: [Zustand/Redux/Jotai/none — detect]
- Data fetching: [SWR/TanStack Query/Server Actions]
- Backend: [Server Actions/API Routes/tRPC/external API]
- Testing: [Vitest/Jest/Playwright]

## Architecture Layers (App Router)
- `app/` — routes (server components by default)
- `app/api/` — API route handlers
- `components/` — reusable UI components
- `lib/` — utilities, clients, shared logic
- `hooks/` — custom React hooks
- `actions/` — server actions (if used)
- `types/` — shared TypeScript types

## Golden Principles
- Server Components by default, `"use client"` only when needed (interactivity, browser APIs)
- Fetch data in Server Components, not client
- Use Server Actions for mutations, not API routes when possible
- Colocate: component + styles + tests together
- No prop drilling — use context or state library
- Loading and error states are FIRST-CLASS (loading.tsx, error.tsx)
- [project-specific]

## Verification Requirements (MANDATORY)
1. `npm run lint` — ESLint must pass
2. `npm run typecheck` — TypeScript strict, no errors
3. `npm test` — unit tests pass
4. `npm run build` — must build without errors
5. `npm run e2e` for UI changes [if Playwright]

If any fail after 2 attempts, ESCALATE.

## Escalation Protocol
Same as above.

## Reviewer Requirements
Two-stage review.

## GC Compliance
Architecture changes update this file.

## Where Things Live
- Decisions: `docs/decisions/`
- Design system: `docs/design/`
- API contracts (if API routes): `docs/api/`

## What NOT to Do
- Never `"use client"` unnecessarily — kills performance
- Never fetch in client component when server component suffices
- Never mutate state directly — always via setState/action
- Never `any` type — use `unknown`
- Never inline styles (use Tailwind/CSS Modules per convention)
- Never large components — extract when > 100 lines
- Never business logic in components — extract to lib/ or actions/
- Never skip metadata for SEO-important pages
- [project-specific]

## Graphify Reality Check
[If applicable]
```

---

## Template: TypeScript (Generic, no framework)

```markdown
# AGENTS.md

## What This Is
[project-specific]

## Stack
- TypeScript [version]
- Runtime: [Node.js/Deno/Bun — detect]
- Package manager: [npm/pnpm/yarn/bun — detect]
- Testing: [Vitest/Jest — detect]
- Linting: [ESLint/Biome — detect]
- Build: [tsc/tsup/esbuild/rollup — detect]

## Architecture Layers
[Detect from src/ structure. Common patterns:]
- `src/core/` — domain logic (pure, no I/O)
- `src/adapters/` — I/O boundary (DB, HTTP, filesystem)
- `src/services/` — orchestration
- `src/utils/` — helpers
- `src/types/` — shared types
- `src/index.ts` — entry point

## Golden Principles
- Strict mode always (`"strict": true` in tsconfig)
- No `any` — use `unknown` and narrow
- Pure functions in core/, side effects at edges
- Explicit return types on exported functions
- Zod (or similar) for runtime validation at boundaries
- [project-specific]

## Verification Requirements (MANDATORY)
1. Lint check
2. Type check (strict, no errors)
3. Tests pass
4. Build succeeds

Fail 2x → ESCALATE.

## Escalation, Reviewer, GC — same as above templates

## Where Things Live
- Decisions: `docs/decisions/`

## What NOT to Do
- Never `any` — use `unknown`
- Never suppress TS errors with `@ts-ignore` — use `@ts-expect-error` with reason
- Never mutate function parameters
- Never side effects in constructor
- [project-specific]
```

---

## Template: PHP (non-Laravel)

```markdown
# AGENTS.md

## What This Is
[project-specific]

## Stack
- PHP [version from composer.json]
- Framework: [Symfony/Slim/CodeIgniter/none — detect]
- Testing: [PHPUnit/Pest]
- Static analysis: [PHPStan/Psalm]
- Code style: [PHP-CS-Fixer/Pint]
- Database: [detected]

## Architecture Layers
[Detect from src/ structure]

## Golden Principles
- Type declarations everywhere (`declare(strict_types=1);`)
- No global state — dependency injection
- Value objects for domain concepts (not arrays)
- Repository pattern for data access
- Services for business logic
- [project-specific]

## Verification Requirements (MANDATORY)
1. `vendor/bin/phpunit` or `vendor/bin/pest`
2. `vendor/bin/phpstan analyse`
3. `vendor/bin/php-cs-fixer fix --dry-run`

Fail 2x → ESCALATE.

## Escalation, Reviewer, GC — standard

## What NOT to Do
- Never `var_dump()` or `print_r()` in committed code
- Never mixed types where specific type works
- Never `@` error suppression
- Never `include` at runtime for logic (use autoload)
- Never bypass strict types
- [project-specific]
```

---

## Template: Python

```markdown
# AGENTS.md

## What This Is
[project-specific]

## Stack
- Python [version from pyproject.toml or requirements.txt]
- Package manager: [uv/poetry/pip — detect]
- Framework: [FastAPI/Django/Flask/none — detect]
- Testing: [pytest — detect version]
- Linting: [Ruff/Flake8+Black]
- Type checking: [mypy/pyright/basedpyright]
- Database: [SQLAlchemy/Django ORM/raw]

## Architecture Layers
[Detect from src/ or package structure]

Common patterns:
- `src/domain/` — business entities (pure)
- `src/application/` — use cases
- `src/infrastructure/` — I/O adapters (DB, HTTP)
- `src/api/` or `src/routers/` — HTTP endpoints
- `src/models/` — Pydantic models or SQLAlchemy entities

## Golden Principles
- Type hints on all function signatures (verified by mypy/pyright)
- Pydantic (v2) for data validation at boundaries
- Async where I/O bound, sync where CPU bound
- Repository pattern for data access
- Dependency injection via FastAPI Depends or explicit constructor
- [project-specific]

## Verification Requirements (MANDATORY)
1. `ruff check .` — must pass
2. `ruff format --check .` — must pass
3. `mypy .` [or `pyright`] — must pass strict
4. `pytest` — all tests pass
5. `pytest --cov` if coverage tracked

Fail 2x → ESCALATE.

## Escalation, Reviewer, GC — standard

## What NOT to Do
- Never `print()` in production code — use logging
- Never bare `except:` — catch specific exceptions
- Never mutable default arguments (`def f(x=[])`) — use `None` and initialize
- Never `import *` — explicit imports only
- Never skip type hints on public functions
- Never `Any` type unless truly opaque — use TypeVar or Protocol
- [project-specific]
```

---

## Template: Rust / Tauri

Use when stack includes Rust, especially Tauri desktop apps. If a frontend framework is present (Next.js, Vite/React, Svelte), generate a root `AGENTS.md` for the Rust/Tauri shell and a per-frontend `AGENTS.md` using the matching frontend template.

```markdown
# AGENTS.md

## What This Is
[project-specific: 1-2 sentences describing the desktop app / service]

## Stack
- Rust [edition from Cargo.toml — e.g. 2021]
- Shell: [Tauri 2 / plain binary / axum / actix — detect]
- Frontend (if Tauri): [Next.js static export / Vite+React / SvelteKit — detect]
- Async runtime: [tokio — from Cargo.toml]
- Database: [sqlx/rusqlite/diesel — detect]
- Serialization: [serde/serde_json]
- Error handling: [thiserror + anyhow]
- Testing: [`cargo test`]
- Linting: [`cargo clippy`, `cargo fmt`]

## Architecture Layers
[If Graphify available, use actual detected layers. Otherwise use these conventions:]
- `src-tauri/src/commands/` — `#[tauri::command]` thin API surface (no business logic)
- `src-tauri/src/domain/` — entities + state machines (pure, no I/O)
- `src-tauri/src/services/` — business logic, external integration
- `src-tauri/src/storage/` — repositories, migrations, atomic fs helpers
- `src-tauri/src/events.rs` — Tauri event emit helpers
- `src-tauri/migrations/` — SQL migrations (versioned, never edited after merge)
- `src/` — frontend (separate AGENTS.md if non-trivial)
- `packages/bindings/` — generated Rust→TS types (never hand-edit)

## Golden Principles
- Commands are thin: parse input → call service → return DTO. No business logic in `commands/`.
- One writer: SQLite + filesystem touched only via `storage/`. Frontend never accesses DB/FS directly.
- `src-tauri/src/domain/` and `harness/` must not call Tauri APIs — keep them testable without a runtime.
- Errors are typed: `Result<T, AppError>` with `thiserror`; `anyhow` only at boundaries. Never `unwrap()`/`expect()` in request paths.
- All external paths canonicalized before use.
- Generated bindings are the only contract between Rust and TS — regenerate, don't hand-write.
- Frontend is presentational: data only via `invoke`/`listen`.
- [project-specific conventions]

## Verification Requirements (MANDATORY)
Before marking any task complete, agent MUST run:
1. `cargo fmt --check` — formatting
2. `cargo clippy --all-targets -- -D warnings` — no lints
3. `cargo test` — all Rust tests pass
4. Frontend (if present): `pnpm lint && pnpm typecheck && pnpm test`
5. Frontend build: `pnpm build` (must succeed, incl. static export if applicable)

If ANY of these fail after 2 fix attempts, ESCALATE — do NOT proceed.

## Escalation Protocol
When stuck after 2 attempts:
1. Write escalation to `.agent-workspace/escalations/[task-id].md`
2. Include: what failed, exact error, what you tried, what you suspect
3. STOP — do not proceed to next task
4. Human intervention required before continuing

## Reviewer Requirements
Two-stage review before merge:
1. **Spec-compliance reviewer:** Does implementation match the approved spec?
2. **Code-quality reviewer:** Safety, ownership, error handling, concurrency, perf
Both must pass. Reviewers are separate concerns — don't combine.

## GC Compliance
If you change architecture (add layer, new pattern, refactor):
- Update THIS AGENTS.md in the SAME PR
- Update relevant `docs/architecture/` files
- Never let this file drift from actual implementation
Weekly GC scan will detect drift and create issues.

## Where Things Live
- Decisions: `docs/decisions/`
- API/bindings contracts: `docs/api/` (or generated bindings)
- Migrations: `src-tauri/migrations/`
- Runbooks: `docs/runbooks/`

## What NOT to Do
- Never `unwrap()`/`expect()`/`panic!()` in request or command paths — propagate errors
- Never block the async runtime — use async I/O (`tokio`), not sync calls in async fns
- Never write to DB/FS outside `storage/`
- Never edit applied migrations — add a new versioned migration
- Never hand-edit generated bindings — regenerate
- Never `unsafe` without a `// SAFETY:` comment and reviewer approval
- Never clone large data to sidestep borrow-checker — fix ownership
- Never `#[allow(clippy::...)]` to silence a real issue
- Never expose business logic in `commands/`
- [project-specific anti-patterns]

## Graphify Reality Check
[If applicable]
```

---

## Variant: Next.js Static Export (Tauri desktop shell)

When the Next.js project runs inside a Tauri/webview shell with `output: 'export'` (no Node server), the standard Next.js template above is misleading. Adjust:

- **No Server Components / Server Actions / API routes** — everything is client-side. Backend calls go through Tauri `invoke`/`listen`, not fetch to a Next server.
- **No dynamic route segments** (`app/[id]/page.tsx`) — static export can't enumerate runtime data. Use static routes + query params (`/project?id=…`) with `useSearchParams` + Suspense.
- **Verification** adds a Rust gate: `cargo fmt --check`, `cargo clippy -D warnings`, `cargo test` (if `src-tauri/` present).
- **"Data fetching"** = Tauri commands (wrapped in TanStack Query), not SWR/Server Actions.
- **"Never `use client` unnecessarily"** does NOT apply — the whole app is client-rendered; prefer explicit client components.
- Contract rule: Rust command/event types are generated (`tauri-specta`) into a bindings package; never hand-write `invoke` string names.

## Multi-Stack Combination

For projects with multiple stacks (e.g., Laravel backend + Next.js frontend in monorepo):

1. Generate ONE `AGENTS.md` at monorepo root that:
   - Describes overall architecture (both stacks)
   - Lists repos with their stack
   - Cross-cutting principles (shared conventions)
   - References per-repo AGENTS.md for stack-specific detail

2. Generate per-repo `AGENTS.md` in each sub-repo using single-stack template

Root AGENTS.md example structure:

```markdown
# AGENTS.md — [Project Name] Monorepo

## What This Is
[Overall system description]

## Repositories
- `apps/api/` — Laravel backend (see `apps/api/AGENTS.md`)
- `apps/web/` — Next.js frontend (see `apps/web/AGENTS.md`)
- `apps/admin/` — Filament admin (see `apps/admin/AGENTS.md`)

## Cross-Cutting Principles
- API contracts defined in `docs/api/openapi.yaml` — never modify without updating docs
- Shared types in `packages/shared-types/` — regenerate frontend types after API changes
- Auth: JWT with refresh, session in HttpOnly cookie
- [project-specific]

## Verification (Monorepo-wide)
- Run per-repo verification (see each AGENTS.md)
- Additionally: `pnpm test:integration` at root for cross-repo contract tests

## Escalation, Reviewer, GC — standard, plus:
- Cross-repo changes require reviewer from each affected repo

## Where Things Live
- API contracts: `docs/api/`
- Shared decisions: `docs/decisions/`
- Per-repo details: `apps/*/AGENTS.md`, `apps/*/docs/`
```

---

## Quality Checks Before Writing

Before writing AGENTS.md, verify:

- [ ] Length ≤ 120 lines (target 80-100)
- [ ] All required sections present: What This Is, Stack, Architecture Layers, Golden Principles, Verification Requirements, Escalation Protocol, Reviewer Requirements, GC Compliance, Where Things Live, What NOT to Do
- [ ] Verification commands are runnable (correct paths, real commands)
- [ ] Golden Principles are specific to project (not generic "write clean code")
- [ ] What NOT to Do has concrete examples
- [ ] Architecture Layers references actual paths that exist in the project
- [ ] If Graphify was used, "Graphify Reality Check" section included
- [ ] If stack is Next.js inside a Tauri/static-export shell, the static-export variant rules are applied (no Server Components/Actions/API routes, no dynamic segments)
- [ ] If stack is Rust/Tauri, commands stay thin and DB/FS access is confined to `storage/`
- [ ] No placeholder text like `[project-specific]` remaining — all filled

If any check fails, revise before writing.

---

## Improvement Mode (Existing AGENTS.md)

When `mode: improve` or existing AGENTS.md detected:

1. Read existing AGENTS.md
2. Read Graphify report (if available)
3. Identify gaps:
   - Missing harness sections (Verification, Escalation, etc.)?
   - Outdated stack info?
   - Drift from actual architecture (per Graphify)?
   - Missing conventions user mentioned in context prompt?
4. **Preserve** user's existing custom content (don't overwrite)
5. **Add** missing sections
6. **Update** outdated sections with note: `<!-- Updated by Kekang on [date]: [reason] -->`
7. Show diff to user before applying (via Kekang UI) or write directly and report changes

---

## Anti-Patterns to Avoid in Generated Output

- ❌ "Follow SOLID principles" — too abstract, agents can't verify
- ❌ "Write clean code" — meaningless
- ❌ Copy-paste template verbatim without customization
- ❌ Listing every file in the project (that's what tree does)
- ❌ Marketing-speak ("scalable, robust, enterprise-grade")
- ❌ Aspirational content (things project *should* do but doesn't)
- ✅ Concrete rules that lint/tests/reviewer can enforce
- ✅ References to actual files as examples
- ✅ Anti-patterns that are common in this specific stack/project
- ✅ Reality Check section when Graphify contradicts assumption

---

## Skill Metadata

- **Version:** 0.2.0
- **Compatible with:** Claude Code (all recent versions), harness engineering workflow, Kekang v2.0 (Rust + Tauri + Next.js)
- **Depends on (optional):** Graphify (`graphifyy` on PyPI) for grounded architecture
- **Installed by:** Kekang app auto-install, or manual copy to `~/.claude/skills/kekang-generate-agents/`
- **Invoked by Kekang via:** a managed `claude` CLI session (`ClaudeCliService`), not a detached subprocess

## Installation Manual (without Kekang)

```bash
mkdir -p ~/.claude/skills/kekang-generate-agents
# Copy this file to:
# ~/.claude/skills/kekang-generate-agents/SKILL.md

# Then in Claude Code:
# /kekang:generate-agents
```

## Feedback / Issues

Report to Kekang project or improve locally. This skill is standalone and can be modified per user preference.