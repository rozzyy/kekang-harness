# Kekang

**Desktop harness engineering dashboard for Claude Code workflows**

A personal desktop app that enforces harness engineering discipline across multi-project, multi-profile Claude Code development. Ships no Python/Node runtime — pure Rust backend drives the `claude` CLI over the stream-json control protocol.

```text
┌─ Projects ─────────────────────────────────────────────────┐
│  📁 my-laravel-project         [PRIBADI]                  │
│     Phase: 2/5 (Execution in progress)                    │
│     Bootstrap: ✅ 6/6                                    │
│     Health: 🟢 healthy                                    │
│                                                              │
│  📁 nestjs-api-kantor           [KANTOR]                  │
│     Phase: 1/5 (Bootstrap incomplete)                     │
│     Bootstrap: ⚠️ 3/6 — missing linter, hooks            │
└────────────────────────────────────────────────────────────┘
```

[![Status](https://img.shields.io/badge/status-ready%20for%20implementation-blue?style=flat-square)](https://github.com/your-org/kekang)
[![Stack](https://img.shields.io/badge/Stack-Rust%20%7C%20Tauri%202%20%7C%20Next.js%2015%20%7C%20SQLite-purple?style=flat-square)](#)
[![License](https://img.shields.io/badge/License-Personal%20Use%20Only-red?style=flat-square)](#)

---

## Features

### 5-Phase Harness Workflow

Enforce discipline consistently across every project:

1. **Bootstrap** — Detect stack, generate AGENTS.md, scaffold docs/, install linters + hooks
2. **Brainstorm → Plan → Execute** — Structured workflow with explicit human approval gates
3. **Verification Loop** — Pre-commit hooks, retry policy, two-stage review (spec → code)
4. **Garbage Collection** — Weekly scan via Graphify: god nodes, drift detection, boundary violations
5. **Observability** — Cost per verified outcome, retry/escalation rates, retro capture

### Integrated Chat Interface

Prompt Claude Code directly from the dashboard. Same `claude` CLI you already use — no API key, no separate runtime.

- Real-time streaming with inline approve/deny for tool calls
- Session resume, fork, export
- Budget tracking (80%/100% warnings)
- Multi-project side-by-side support

### Multi-Project Dashboard

Phase state, health status, skills status across all projects at a glance. File watcher auto-refreshes on `.agent-workspace/` changes.

### Subagent Configuration

Form-based model/tool/permission editor per project. Built-in presets:
- **Harness Standard** — Opus planner, Sonnet executor, Sonnet reviewers
- **Cost-optimized** — Opus planner only, Sonnet everywhere else, Haiku search/doc
- **Quality-first** — Opus everywhere

### Tool & Skill Manager

One-stop UI to install, configure, toggle: Graphify, Superpowers, RTK, Caveman, Ponytail, Context7 MCP.

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| App shell | Tauri 2 |
| Backend | Rust (tokio, sqlx, notify) |
| Frontend | Next.js 15 (App Router, `output: 'export'`) |
| UI | Tailwind v4 + shadcn/ui (dark-only, primary accent `#7C3AED`) |
| Storage | SQLite via sqlx |
| Claude integration | `claude` CLI over stream-json control protocol |
| Bindings | tauri-specta + specta |

**No Python/Node runtime shipped.** All AI calls via Claude Code subscription.

---

## Installation

### Prerequisites

| Tool | Version |
|------|---------|
| Rust | ≥ 1.80 stable |
| Node.js | 20+ |
| pnpm | 9+ |
| claude CLI | latest (`brew install --cask claude-code`) |

### Dev Setup

```bash
# Clone
git clone https://github.com/your-org/kekang.v2
cd kekang.v2

# Install deps
pnpm install

# Frontend only (hot reload at localhost:3000)
pnpm dev

# Full Tauri app
cd apps/desktop && cargo tauri dev
```

### Build

```bash
pnpm build          # Next.js static export
cargo tauri build   # Tauri binary + bundlers
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Kekang App (Tauri 2)                      │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Next.js (output: 'export', webview)           │  │
│  └──────────────────────┬─────────────────────────────────┘  │
│  ┌──────────────────────▼─────────────────────────────────┐  │
│  │  commands/ — #[tauri::command] wrappers, no logic      │  │
│  └──────────────────────┬─────────────────────────────────┘  │
│  ┌──────────────────────▼─────────────────────────────────┐  │
│  │  harness/ — PhaseTracker, WorkflowEnforcer, Bootstrap   │  │
│  └──────────────────────┬─────────────────────────────────┘  │
│  ┌──────────────────────▼─────────────────────────────────┐  │
│  │  services/ — ClaudeCliService, ChatSession, Watcher, GC  │  │
│  └──────────────────────┬─────────────────────────────────┘  │
│  ┌──────────────────────▼─────────────────────────────────┐  │
│  │  storage/ — sqlx repos, atomic fs ops, notify           │  │
│  └──────────────────────┬─────────────────────────────────┘  │
└──────────────────────────┼───────────────────────────────────┘
```

**Golden rules:**
- Rust is the single writer of SQLite, filesystem, and child processes
- Frontend never touches DB/FS — all via Tauri `invoke`/`listen`
- `claude/protocol.rs` isolates all CLI wire format knowledge

---

## Project Structure

```
kekang.v2/
├── apps/desktop/
│   ├── src/               # Next.js frontend
│   │   ├── app/           # App Router pages
│   │   ├── components/    # UI components
│   │   └── lib/           # Tauri bindings
│   └── src-tauri/         # Rust backend
│       ├── src/
│       │   ├── commands/  # #[tauri::command] wrappers
│       │   ├── harness/   # Phase tracking, workflow enforcement
│       │   ├── services/  # Business logic + external integration
│       │   ├── claude/    # CLI process + stream-json protocol
│       │   └── storage/   # sqlx repositories, migrations
│       └── Cargo.toml
├── packages/bindings/     # Generated Rust→TS types (specta)
└── docs/
    ├── SPEC.md           # Full specification (source of truth)
    ├── DESIGN.md         # Design system tokens from Stitch
    └── Kekang-generate-agent-skill.md
```

---

## Design System

UI built with Google Stitch — "Kekang Developer Harness Dashboard"

- **Primary accent:** `#7C3AED`
- **Mode:** Dark-only
- **Fonts:** Inter (UI), JetBrains Mono (code/telemetry)
- **Full tokens:** [`docs/DESIGN.md`](docs/DESIGN.md)
- **Stitch project:** `projects/13020582949250899293`

---

## Verification

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
pnpm lint
pnpm typecheck
pnpm test
pnpm build
```

---

## Status

**v2.0** — Rust + Tauri 2 + Next.js. Feature-equivalent to v1.1 (Python + Flet). Ready for implementation.

Full specification: [`docs/SPEC.md`](docs/SPEC.md)

---

## Author

**Rozzy Rahmanda** — [tanyadev.id](https://tanyadev.id)
