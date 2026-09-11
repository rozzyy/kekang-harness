# AGENTS.md

## What This Is

Kekang — a personal desktop dashboard that enforces harness-engineering discipline for Rozzy's
multi-project, multi-profile Claude Code workflow. Tauri 2 shell: **Rust backend, Next.js frontend**.

Read `docs/SPEC.md` first. It is the source of truth for scope, architecture, and data model.

## Stack

- Shell: Tauri 2
- Backend: Rust (tokio, sqlx/SQLite, notify, `tokio::process`)
- Frontend: Next.js 15 (App Router, `output: 'export'`), TypeScript, Tailwind v4 + shadcn/ui
- Claude integration: `claude` CLI over stream-json control protocol (no Agent SDK runtime)
- Bindings: `tauri-specta` + `specta`
- Package manager: pnpm (workspace)

## Source of Truth

| Topic | File |
|---|---|
| Scope, architecture, data model, timeline | `docs/SPEC.md` |
| UI design system + Stitch reference | `docs/DESIGN.md` |
| `/kekang:generate-agents` skill | `docs/Kekang-generate-agent-skill.md` |

## UI Design Reference (MANDATORY)

All UI/frontend work MUST follow the Kekang design in **Google Stitch** — do not invent visual
language, tokens, or layouts.

- **Stitch project:** "Kekang Developer Harness Dashboard" — `projects/13020582949250899293`
- **MCP server:** `stitch` (remote, configured in opencode). Restart opencode after config changes.
- **Local mirror:** `docs/DESIGN.md` (tokens: colors, typography, spacing, radius, components).
- **Screens:** 15 desktop screens + logo/asset, listed in `docs/DESIGN.md`.

Rules:
1. Before building any screen/component, fetch the matching Stitch screen (`get_screen` /
   `list_screens`) and match its layout, tokens, density, and states.
2. Use only tokens from `docs/DESIGN.md`. No ad-hoc colors, fonts, radii, or spacing.
3. If a screen does not exist in Stitch, generate/confirm it in Stitch first
   (`generate_screen_from_text` / `edit_screens`), then mirror tokens back into `docs/DESIGN.md`.
4. When Stitch and `docs/DESIGN.md` disagree, Stitch wins — update the mirror.
5. Dark mode only (`colorMode: DARK`). Primary accent `#7C3AED`. Fonts: Inter (UI), JetBrains Mono (code/telemetry).

## Architecture Layers

Full layout in `docs/SPEC.md` §6.5/§8. Summary:
- `apps/desktop/src/` — Next.js UI (presentational only; data via Tauri `invoke`/`listen`)
- `apps/desktop/src-tauri/src/commands/` — thin `#[tauri::command]` surface, no business logic
- `apps/desktop/src-tauri/src/harness/` — phase tracking, workflow enforcement, state validation, bootstrap
- `apps/desktop/src-tauri/src/services/` — business logic + external integration
- `apps/desktop/src-tauri/src/claude/` — `claude` CLI process + stream-json control protocol
- `apps/desktop/src-tauri/src/storage/` — sqlx repositories, migrations, atomic filesystem ops
- `packages/bindings/` — generated Rust→TS types (never hand-edit)

## Golden Principles

- Rust is the single writer of SQLite, filesystem, and child processes. Frontend never touches DB/FS.
- Commands are thin: parse → call service → return DTO.
- `harness/` and `domain/` must not call Tauri APIs — keep them testable without a runtime.
- Never `unwrap()`/`expect()`/`panic!()` in request paths; typed errors (`thiserror`) + `anyhow` at boundaries.
- `claude/protocol.rs` is the only place that knows the CLI wire format — isolate all CLI drift there.
- Generated bindings are the only Rust↔TS contract — regenerate, never hand-write `invoke` names.
- Atomic writes: temp → fsync → rename. Back up before overwrite.

## Verification Requirements (MANDATORY)

Run before claiming any task complete (once scaffold exists):

1. `cargo fmt --check`
2. `cargo clippy --all-targets -- -D warnings`
3. `cargo test`
4. `pnpm lint`
5. `pnpm typecheck`
6. `pnpm test`
7. `pnpm build` (must produce a valid static export)

If ANY fails after 2 fix attempts, ESCALATE — do not proceed.

## Escalation Protocol

When stuck after 2 attempts:
1. Write escalation to `.agent-workspace/escalations/[task-id].md`
2. Include: what failed, exact error, what you tried, what you suspect
3. STOP — do not proceed to the next task
4. Human intervention required before continuing

## Reviewer Requirements

Two-stage review before merge:
1. **Spec-compliance** — does the change match `docs/SPEC.md`?
2. **Code-quality** — safety, ownership, error handling, concurrency, perf, UI-token compliance

Both must pass. Keep the two reviews separate.

## GC Compliance

Architecture/design changes update the affected docs in the SAME change:
- Architecture → `docs/SPEC.md` (+ `docs/architecture/` once it exists)
- UI tokens/screens → `docs/DESIGN.md` and the Stitch project
Never let docs drift from implementation.

## Where Things Live

- Spec: `docs/SPEC.md`
- Design: `docs/DESIGN.md`
- Skill: `docs/Kekang-generate-agent-skill.md`
- Migrations: `apps/desktop/src-tauri/migrations/` (once created)
- Generated bindings: `packages/bindings/`
- Workspace state/handoffs: `.agent-workspace/` (gitignored)

## What NOT to Do

- Never invent UI, colors, or layouts outside the Stitch design / `docs/DESIGN.md`.
- Never access SQLite/filesystem from the frontend — go through Rust commands.
- Never ship a Python or Node runtime — backend is Rust, frontend is static export.
- Never edit applied migrations — add a new versioned migration.
- Never hand-edit generated bindings.
- Never block the async runtime; use tokio async I/O.
- Never commit secrets. The Stitch API key lives in `~/.config/opencode/secrets/stitch.key` (gitignored), referenced via `{file:...}`.
- Never bypass the 5-phase workflow or skip verification gates.
