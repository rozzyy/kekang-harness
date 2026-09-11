# Kekang — Specification Document v2.0

**Personal Harness Engineering Dashboard untuk Rozzy's Claude Code workflow**

> **Supersedes v1.1.** Major change: full stack swap from Python + Flet to **Next.js + Rust + Tauri 2**, with the **`claude` CLI driven directly over the stream-json control protocol** (no Python/Node Agent SDK runtime shipped). Feature scope unchanged — the entire v1.1 feature set (5-phase workflow, integrated chat, bootstrap, GC, metrics) is preserved.

---

## Metadata

| Field | Value |
|-------|-------|
| Version | 2.0 (Rust + Tauri stack) |
| Author | Rozzy Rahmanda (tanyadev.id) |
| Status | Ready for implementation |
| Timeline MVP | 7 weeks (incl. Week 0 protocol spike) |
| Target user | Personal (single-user) |
| Stack | Rust + Tauri 2 + Next.js 15 + SQLite (sqlx) + `claude` CLI (stream-json control protocol) |
| Distribution | Local desktop app, tidak untuk dijual |
| AI dependency | Claude Code subscription (via `claude` CLI, no separate API key) |
| UI design | Google Stitch — "Kekang Developer Harness Dashboard" (`projects/13020582949250899293`), mirror di `docs/DESIGN.md` |

---

## TL;DR

Kekang adalah desktop app yang **enforce harness engineering discipline** untuk workflow Claude Code Rozzy across multi-project, multi-repo, dan multi-Claude-profile. Core value: bikin kerjaan coding lebih **reliable** (bukan cuma cepat) dengan bootstrap otomatis 5-phase harness workflow (OpenAI + Anthropic combined approach), grounded di real code understanding via Graphify. **Rozzy prompt Claude Code langsung dari Kekang** via integrated chat — no context switch antara Kekang dashboard dan terminal.

**Yang Kekang lakukan:**
- **Integrated chat interface** — prompt Claude Code langsung dari Kekang UI, drive `claude` CLI via stream-json control protocol
- Bootstrap project ke harness-ready state (AGENTS.md, docs/, linter, hooks, subagents)
- Generate AGENTS.md via custom skill + Graphify graph data
- Manage 5-phase feature workflow (brainstorm → plan → execute → verify → retro)
- Monitor escalation cross-project cross-profile
- Configure subagent model per project (Opus/Sonnet/Haiku per role)
- Install & manage token-saver tools (RTK, Caveman, Ponytail, Graphify, Superpowers, Context7)
- Schedule GC scan untuk drift detection

**Yang Kekang TIDAK lakukan:** Cloud sync, multi-user, replace Claude Code engine (just wrap it), reinvent tool loop, custom API integration.

**Note tentang "chat interface":** Ini bukan re-implementasi Claude Code — Kekang drives process `claude` yang sama persis dengan CLI, lewat protokol stream-json bidirectional yang sama yang dipakai Agent SDK resmi. Skills, subagents, MCP servers, slash commands semua works. Yang berubah cuma rendering: dari terminal ke web UI (Next.js) dengan approve/deny buttons untuk tool calls.

**Arsitektur singkat:** Tauri 2 desktop shell. **Rust = seluruh backend** (SQLite, filesystem, process, watcher, scheduler, Claude CLI driver). **Next.js = seluruh frontend** (static export, presentational, semua data via Tauri `invoke` + events). Gak ada Python/Node runtime yang di-ship — cuma binary Tauri + `claude` CLI yang memang sudah dipakai.

---

## Table of Contents

1. [Problem Statement & Positioning](#1-problem-statement--positioning)
2. [Core Philosophy](#2-core-philosophy)
3. [The 5-Phase Harness Workflow](#3-the-5-phase-harness-workflow)
4. [Supporting Features](#4-supporting-features)
5. [Non-Goals](#5-non-goals)
6. [Technical Architecture](#6-technical-architecture)
7. [Data Model](#7-data-model)
8. [Directory Structure](#8-directory-structure)
9. [Custom Skill: `/kekang:generate-agents`](#9-custom-skill-kekanggenerate-agents)
10. [Implementation Timeline](#10-implementation-timeline)
11. [Success Criteria](#11-success-criteria)
12. [Open Questions & Recommendations](#12-open-questions--recommendations)
13. [Risks & Mitigation](#13-risks--mitigation)
14. [Prerequisites & Setup](#14-prerequisites--setup)
15. [Post-MVP Roadmap](#15-post-mvp-roadmap)

---

## 1. Problem Statement & Positioning

### Pain Points

Rozzy handle multi-project development dengan Claude Code:
- 2 Claude profile (pribadi + kantor) via native `CLAUDE_CONFIG_DIR`
- 1-3 repo per project, 2+ project paralel
- 3+ Claude Code session concurrent
- 6 stack berbeda: Laravel, NestJS, Next.js, TypeScript, PHP, Python
- Sudah pake Superpowers, Ponytail, punya AGENTS.md manual per project

**Friction utama:**

1. **Setup tool ekosistem tedious per machine baru** — RTK, Caveman, Ponytail, Superpowers, Graphify install manual satu-satu
2. **Toggle skill per project ribet** — edit `.claude/settings.json` manual per project
3. **Bikin AGENTS.md manual per project baru** — 30-60 menit per project untuk 6 stack berbeda
4. **Konfigurasi subagent model tedious** — edit `.claude/agents/*.md` frontmatter manual
5. **Escalation dari session paralel sering ke-miss** — 5 session jalan, satu escalate, gak sadar
6. **Cross-profile confusion** — kadang bingung session pake profile pribadi atau kantor
7. **Cross-project context switching** — mau tau state semua project butuh buka satu-satu
8. **Harness discipline manual gak scale** — dengan 5+ project, susah maintain 5-phase workflow konsisten

### Goal

**Reliability > speed.** Kekang bikin kerjaan coding lebih **konsisten, less error, less miss** — bukan sekadar lebih cepat. Cepat tapi jelek adalah anti-goal.

### Positioning

**Kekang bukan:**
- Product komersial (personal use only)
- Pengganti Claude Code / Codex
- Orchestrator paralel agent runtime (Orkes/Conductor teritori)
- Traffic-layer tool (RTK/Caveman/9Router teritori)
- Cloud SaaS

**Kekang adalah:**
- **Meta-tool** yang enforce harness engineering discipline
- **Orchestrator config & state** across projects
- **Grounded** di real code understanding (via Graphify)
- **Subscription-native** — semua AI call via Claude Code, no API key

---

## 2. Core Philosophy

### Central Thesis

**Reliability datang dari discipline yang grounded di real code understanding.**

- Kekang enforce discipline (5-phase harness workflow)
- Graphify provide truth (knowledge graph dari actual code)
- Claude Code execute (via subscription, no API key)
- Rozzy provide judgment (approval gates, escalation resolution, merge decision)

### Insight Kunci

1. **Persistence is file-based, not session-based.** State di `.agent-workspace/`, config di file, skill di `.claude/`. Discipline > tooling.

2. **Constraints > flexibility.** Reliability datang dari membatasi solution space via linters, hooks, spec enforcement — bukan kebebasan generation.

3. **Verification is the bottleneck.** Generation infinite dan murah; verification finite dan mahal. Invest di verification layer.

4. **Human-in-the-loop at decision points.** Automation di execution, human di approval gates. Fully-automated coding = ship sampah cepat.

5. **Cost per verified outcome > cost per run.** Metric benar mengukur apa yang ship dan gak di-revert.

6. **Grounded > generated.** LLM tanpa real code context = hallucinating with confidence. LLM + graph = reasoning with evidence.

### Combined OpenAI + Anthropic Approach

Kekang implement gabungan:

**OpenAI-style (Static Layer):**
- AGENTS.md sebagai central map
- docs/ structure (architecture, golden-principles, decisions)
- Architectural linters yang enforce invariant
- Pre-commit hooks
- Garbage collection subagent

**Anthropic-style (Dynamic Layer):**
- Superpowers workflow (brainstorm → plan → subagent-driven-execution)
- Handoff files di `.agent-workspace/`
- Subagent roles (planner, executor, spec-reviewer, code-reviewer)
- Verification loop dengan retry threshold + escalation

Kekang orchestrate keduanya via 5-phase workflow di bagian berikut.

---

## 3. The 5-Phase Harness Workflow

Ini core loop Kekang. Setiap project yang di-manage lewat 5 phase ini. Kekang enforce phase compliance dengan **soft warning default, strict mode opt-in**.

### Phase 1: Static Layer Setup (Bootstrap)

**Trigger:** New project registered, atau existing project yang belum bootstrapped.

**Kekang action (Bootstrap Wizard, 6 steps):**

1. **Detect stack** via marker files (composer.json, package.json, pyproject.toml, dll)
2. **Install/verify Graphify** — build graph pertama kali (opsional, bisa skip untuk speed)
3. **Generate AGENTS.md** via custom skill `/kekang:generate-agents` (spawn `claude` session)
4. **Scaffold docs/** — architecture/, golden-principles/, decisions/, runbooks/, api/
5. **Install architectural linter** — per stack config (eslint-plugin-boundaries untuk JS/TS, PHPStan untuk PHP, ruff untuk Python, dll)
6. **Setup pre-commit hooks** — lint + typecheck + tests
7. **Generate subagent definitions** di `.claude/agents/` (planner, executor, spec-reviewer, code-reviewer, gc-agent)
8. **Scaffold `.agent-workspace/`** dengan handoff file templates + `.gitignore` entry

Semua step preview dulu, user approve, atomic write dengan backup.

**Enforcement:**
- Soft warning kalau incomplete: "Project ini bootstrap 4/6 complete. Reliability turun tanpa full setup."
- Refuse advance ke Phase 2 kalau AGENTS.md gak ada (hard block)
- Track completeness di dashboard: "Bootstrap 6/6 ✅"

### Phase 2: Dynamic Layer Workflow (Per Feature)

**Trigger:** Rozzy mau mulai feature baru.

**3-step orchestration:**

**Step 2.1 — Brainstorming (Spec Creation)**
- Kekang UI: form "New Feature" dengan template pertanyaan:
  - Goal (satu kalimat, verifiable)
  - Constraints (apa yang GAK boleh)
  - Acceptance criteria (bukti selesai)
  - Non-goals (scope creep prevention)
- Kekang spawn `claude` session dengan `/superpowers:brainstorming` context loaded
- Output tersimpan di `.agent-workspace/current-spec.md`
- Enforcement: spec must have all 4 sections filled

**Step 2.2 — Plan Approval**
- Kekang detect saat plan file muncul di `.agent-workspace/current-plan.md`
- UI: side-by-side view — spec di kiri, plan di kanan
- Approval gate: Rozzy explicit approve sebelum execution
- Task list dari plan di-import ke `task-queue.json`
- Enforcement: setiap task punya `why` field (bukan cuma `what`)

**Step 2.3 — Subagent-Driven Execution (Monitored)**
- Kekang launch via `/superpowers:subagent-driven-development`
- Real-time state tracking di UI:
  - Current task
  - Retry count
  - Reviewer state (spec-reviewer → code-reviewer)
- Auto-detect escalation → OS notification
- Auto-detect completion → prompt Rozzy untuk merge decision

**Handoff file management:**
- Kekang enforce format (validate struktur file handoff)
- Warning kalau planner skip `why` field
- Warning kalau task queue flat (no dependencies structure)

### Phase 3: Verification Loop (Continuous)

**Trigger:** Setiap task execution.

**Kekang enforcement:**
- Pre-commit hooks harus pass (lint + typecheck + tests)
- Retry threshold: fail 2x → auto-escalate ke planner
- Track retry rate per project (warning kalau > 50%)
- Two-stage review: spec-compliance THEN code-quality (bukan skip salah satu)

**UI:**
- Live loop status di project detail
- Retry counter, escalation counter
- Health banner kalau unhealthy (retry tinggi, drift detected)

### Phase 4: Garbage Collection (Weekly)

**Trigger:** Schedule (default Mon 9am via Rust scheduler) atau manual trigger.

**Kekang action:**
- Ensure Graphify graph fresh (`/graphify . --update` incremental)
- Spawn `claude` session dengan `/kekang:gc-scan` skill
- Skill query Graphify via MCP untuk deteksi:
  - **God nodes** — files terlalu banyak connection (refactor candidate)
  - **Orphan modules** — unreachable code (safe to delete)
  - **AGENTS.md drift** — architecture di docs vs actual graph gak match
  - **Layer boundary violation** — file di layer X import dari layer Y yang seharusnya gak boleh
  - **Dependency cycle** — A → B → A
- Output report di `.agent-workspace/gc-reports/YYYY-MM-DD.md`
- Categorize: `AUTO_FIX` / `NEEDS_REVIEW` / `NEEDS_DECISION`

**Kekang UI:**
- GC report viewer dengan diff highlighting
- Batch approve auto-fixes
- Escalate NEEDS_DECISION items sebagai task baru

**Enforcement:**
- Warning kalau GC gak jalan > 2 minggu
- Soft warning kalau ada NEEDS_DECISION items > 1 minggu

### Phase 5: Observability & Retro (Continuous + Weekly)

**Metrics tracked:**
- Cost per verified outcome (token usage per merged PR)
- Retry rate per subagent role
- Escalation rate
- Spec-code drift (berapa kali spec di-update mid-execution)
- Time from spec approved → PR merged
- GC findings trend (should decrease)
- Graph health (god node count, density)
- Token savings dari RTK/Caveman/Ponytail

**Retro capture UI:**
- Post-feature auto-prompt (skippable):
  - What worked?
  - What didn't?
  - New skill worth codifying?
  - New guardrail worth adding?
- Weekly aggregate → actionable suggestions:
  - "3x retry karena spec ambiguous — improve brainstorm template"
  - "Executor Sonnet fail 40% di project X — try Opus?"
  - "AGENTS.md project Y outdated 5x GC scan — schedule refresh"

**UI:**
- Metric widget per project di dashboard
- In-app weekly digest

---

## 4. Supporting Features

Feature ini SUPPORT the 5 phases, bukan standalone.

### Feature A: Multi-Project Dashboard

Grid layout dengan **phase state per project**:

```
┌─ Projects ────────────────────────────────────────────────┐
│                                                            │
│  📁 my-laravel-project           [PRIBADI]                │
│     Phase: 2/5 (Execution in progress)                    │
│     Bootstrap: ✅ 6/6                                     │
│     Current: task_003 (executor, retry 1/2)               │
│     Health: 🟢 healthy                                    │
│     Skills: RTK ✅ Caveman ✅ Ponytail (Lite) ✅          │
│                                                            │
│  📁 nestjs-api-kantor            [KANTOR]                 │
│     Phase: 1/5 (Bootstrap incomplete)                     │
│     Bootstrap: ⚠️  3/6 — missing linter, hooks, subagents │
│     [Complete Bootstrap]                                   │
│     Health: 🟡 needs setup                                │
│                                                            │
│  📁 nextjs-portfolio             [PRIBADI]                │
│     Phase: 4/5 (GC pending)                               │
│     Last GC: 3 weeks ago ⚠️                               │
│     [Run GC Now]                                          │
│     Health: 🟡 gc-overdue                                 │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

**Health status:**
- 🟢 healthy: bootstrap complete, GC recent, healthy metrics
- 🟡 warning: bootstrap incomplete, GC overdue, retry rate tinggi
- 🔴 unhealthy: multiple issues

**Auto-refresh** via Rust file watcher (`notify`) di `.agent-workspace/` semua project → Tauri event → UI invalidate.

### Feature B: Custom AGENTS.md Generator (via Skill)

Delegate ke custom skill `/kekang:generate-agents` (spec di Section 9).

**Flow:**
1. User klik "Generate AGENTS.md" di Kekang
2. Kekang siapin `.kekang/generate-request.md` dengan stack, context, Graphify path
3. Kekang spawn `claude` session dengan prompt `/kekang:generate-agents` (env `CLAUDE_CONFIG_DIR` = selected profile)
4. Skill baca request, generate AGENTS.md, tulis ke project root
5. Kekang detect file muncul, preview di UI
6. User accept atau regenerate

**Advantages:**
- Zero API key
- Grounded di Graphify (real code understanding)
- Skill reusable standalone (even without Kekang)

### Feature C: Subagent Configuration UI

Auto-detect subagent files di `.claude/agents/`, form-based editing:

Per subagent:
- **Model dropdown:** `sonnet`, `opus`, `haiku`, `fable`, custom full ID
- **Effort dropdown:** `low`, `medium`, `high`, `inherit`
- **Tools checkbox:** Read, Write, Edit, Bash, Grep, Glob, dll
- **Permission mode:** `default`, `acceptEdits`, `bypassPermissions`, `plan`

**Preset templates:**
- **"Harness Standard"** (default) — Opus planner, Sonnet executor, Sonnet spec-reviewer, Sonnet code-reviewer, Sonnet GC
- **"Cost-optimized"** — Opus planner only, Sonnet everywhere else, Haiku search/doc
- **"Quality-first"** — Opus everywhere
- **"Rozzy custom"** — Rozzy's preferred combo

**Validation warnings:**
- Executor pake Opus effort=high → "Expensive combo, are you sure?"
- Reviewer pake Haiku → "May miss subtle issues"
- Planner pake Sonnet → "OK for simple, use Opus for architecture decisions"

Save → update frontmatter, preserve system prompt body di bawah `---`.

### Feature D: Context Management (Native Profile Switching)

Kelola mapping project → Claude profile.

**Register profiles:**
- Profile "Pribadi" → path: `~/.claude-personal/`, warna hijau
- Profile "Kantor" → path: `~/.claude-work/`, warna biru

**Per project, set default profile via dropdown.**

Saat buka chat atau "Open in Claude Code":
- Set env var `CLAUDE_CONFIG_DIR=<profile-path>`
- Spawn `claude` process dengan env

Dashboard tampilkan badge profile per project (visual anti-confusion).

**Edge case:** user buka Claude Code manual (bukan lewat Kekang) → Kekang gak bisa auto-switch. Solusi: sync via config file, show warning kalau detect mismatch.

### Feature E: Escalation Notifications

OS-level notif dari session mana pun, lintas semua project & profile.

**How:**
- Rust watcher (`notify`) pantau `.agent-workspace/escalations/` semua managed project
- File baru appear → parse (task_id, project, reason, severity)
- Trigger OS notification via `tauri-plugin-notification` (cross-platform)
- Klik notif → Kekang jump ke project + highlight task
- Aggregate view "Inbox" — semua escalation pending

**Notification content:**
```
[my-laravel-project] Escalation — Phase 2 Execution
Task: task_003_add_billing_endpoint
Reason: Executor failed 2 retries — Stripe SDK type mismatch
Retry: 2/2 (maxed out)
[Open]  [Re-plan]  [Dismiss]
```

### Feature F: Tool Installer & Skill Manager UI

One-stop UI untuk install, configure, toggle tools + skills tanpa terminal.

**Tab "Tools" (global, machine-wide):**

Tier 1 (Recommended, unlock Kekang features):
- **Graphify** v[detected] — unlock enhanced AGENTS.md gen + smart GC
- **Superpowers** — unlock spec-driven workflow
- **RTK** — baseline token savings, invisible

Tier 2 (Optional, per-project toggle):
- **Caveman** — output compression
- **Ponytail** — YAGNI-first code (default: Lite)
- **Context7 MCP** — library docs

Tier 3 (Experimental, park):
- **Cavemem MCP**
- **Openclaw/Hermes** (future)

**Per tool:**
- Status indicator (active/inactive/not-installed)
- Show install command (user executes; Kekang gak auto-run sudo) — atau one-click untuk pola aman (brew/curl/uv tanpa sudo)
- Config UI (expose important settings)
- Analytics view (untuk RTK: parse `rtk gain --format json`)
- Uninstall option

**Tab "Skills per Project" (kontekstual):**

Auto-detect skills dari:
- Global: `~/.claude/skills/`
- Project-scoped: `<project>/.claude/skills/`
- Custom Rozzy: yang lu tulis manual

Per skill: toggle enable/disable → edit project's `.claude/settings.json`.

**Preset kombinasi (save/apply):**
- "Rozzy Laravel Stack" — RTK + Superpowers + Context7 + Caveman + Ponytail Lite + laravel-review + supabase-rls-pattern
- "Rozzy Learning Mode" — RTK + Superpowers + Context7 (tanpa compression skill, biar Claude explain detail)
- "Kantor Compliance" — RTK + Superpowers + Caveman (concise) + Ponytail Full

### Feature G: Graphify Manager

**UI (project detail → tab "Knowledge Graph"):**

```
┌─ Graphify for: my-laravel-project ────────────────┐
│  Status: ✅ Installed (v0.8.2)                     │
│  Last graph build: 2 hours ago                    │
│                                                    │
│  Graph stats:                                     │
│  - 342 nodes across 47 files                      │
│  - 891 edges (752 EXTRACTED, 139 INFERRED)        │
│  - Languages: PHP (78%), Blade (15%), JS (7%)    │
│  - 3 god nodes detected                           │
│                                                    │
│  [Rebuild Graph]  [Update (incremental)]          │
│  [Open Interactive Graph]  [View Report]          │
│  [Configure MCP]                                  │
│                                                    │
│  MCP Server:                                      │
│  ✅ Registered in .mcp.json                        │
│  ✅ Available to Claude Code                       │
└────────────────────────────────────────────────────┘
```

**Behavior:**
- Install: `uv tool install graphifyy` via Rust `Command` (show command first)
- Rebuild: `/graphify .` (fresh)
- Update: `/graphify . --update` (incremental, fast)
- Deep mode: `/graphify . --mode deep` (thorough, mahal quota)
- Open graph.html di browser default (Tauri opener plugin)
- Auto-register MCP di project's `.mcp.json`

**Auto-integration:**
- Phase 1 Bootstrap: include Graphify build sebagai optional step
- Phase 2 Feature: suggest update sebelum brainstorming
- Phase 4 GC: primary source of truth
- Phase 5 Metrics: track graph health over time

### Feature H: GC Scheduler

- Global schedule (default Mon 9am) via Rust async scheduler, persisted di `gc_schedules`
- Per-project override
- Manual trigger button
- Report viewer dengan action (batch auto-fix, escalate to tasks)

### Feature I: Metrics Dashboard

Aggregate view dari observability data.

**Per project widget:**
- Cost per merged PR (30-day trend)
- Retry rate (target < 30%)
- Escalation rate (target < 20%)
- Spec-code drift count
- Time-to-merge (median)
- Token savings breakdown

**Global view:**
- Metrics aggregated across all projects
- Weekly digest
- Retro suggestions berdasar pattern

### Feature J: Retro Capture

Post-feature guided form (auto-prompt, skippable):

- What worked?
- What didn't?
- New skill worth codifying?
- New guardrail worth adding?

Monthly aggregate → extract patterns ke skills library.

### Feature K: Integrated Chat Interface

**The primary interaction surface for Kekang.** Rozzy ngeprompt Claude Code langsung dari Kekang tanpa buka terminal external.

**Architecture:**
- Rust spawn satu proses `claude` long-lived per chat session
- Drive protokol stream-json bidirectional (input + output) yang sama dipakai Agent SDK resmi
- CLI yang dipakai = `claude` CLI system (`brew install --cask claude-code` atau setara) — no bundled runtime
- Auth inherit dari `CLAUDE_CONFIG_DIR` (subscription-based, no API key)

**UI Layout (Chat Panel):**

Chat panel is a dedicated UI region in project detail view. Modes:
- **Docked right** (default) — chat panel takes right 40-50% of window, other panels on left
- **Full-screen** — chat expands to fill window (untuk deep work sessions)
- **Minimized** — collapse to sidebar icon, click to expand

**Chat panel components:**

1. **Session header** — session name, model indicator (Sonnet/Opus/Haiku), token counter, cost estimate
2. **Message stream** — scrollable conversation history:
   - User messages (aligned right, muted background)
   - Claude responses (aligned left, streaming with typewriter effect)
   - Tool calls (collapsed cards with expand-on-click, show tool name + args + result)
   - Slash command invocations (chip style, e.g., "`/superpowers:brainstorming`")
   - Subagent handoffs (visual timeline showing planner → executor → reviewer)
3. **Approval prompts** — inline cards when CLI emits a `can_use_tool` control request:
   - Tool name + args preview (syntax highlighted)
   - Buttons: "Approve", "Approve Always (this session)", "Deny", "Approve & Modify" (opens edit modal)
   - Auto-approve toggle per tool type
4. **Input area** — bottom of panel:
   - Multi-line text input dengan Markdown preview
   - Slash command autocomplete (fuzzy match dari installed skills + built-in commands)
   - File attachment button (attach code files ke context)
   - Send button + keyboard shortcut hint (Cmd+Enter)
   - Model selector inline (change model mid-session)
5. **Session controls** — top-right toolbar:
   - "New Session" button
   - "Resume Session" dropdown (list of recent sessions untuk this project)
   - "Fork Session" (branch conversation)
   - "Export Transcript" (Markdown export)
   - "Open in External Terminal" (fallback ke `claude --resume <uuid>` di terminal native)

**Permission Mode Configuration:**

Per-project settings di UI:
- Default mode: `default` (all tool calls require approval)
- Alternatives: `acceptEdits` (auto-approve file edits), `plan` (planning mode), `bypassPermissions` (WARNING banner shown)
- Per-tool allowlist: skip approval untuk specific tools (Read, Grep) sambil require untuk destructive tools (Bash, Write)

**Session Persistence:**

- Sessions tracked via CLI session UUID
- Kekang tracks sessions in `chat_sessions` table + `chat_messages`
- Resume dari Kekang UI → Rust respawn `claude --resume <uuid>`, continues where left off
- Sessions can be resumed even after Kekang restart

**Cost Controls:**

- Set `--max-budget-usd` per session (default $5, configurable)
- Set `--max-turns` (default unlimited, configurable)
- Live cost meter updates as tokens consumed (dari `result` message)
- Warning banner at 80% budget, hard stop at 100%

**Multi-Project Chat:**

- Each project has independent chat sessions
- Switching project auto-switches session context (`cwd` = project path)
- Multi-session paralel: bisa buka 2 project side-by-side dengan chat panel masing-masing
- Session tab bar at top of chat panel (browser-tab style)

**Fallback Mode: External Terminal**

Kalau butuh full TUI experience (checkpoint/rewind, complex visual rendering, unsupported edge cases):
- Click "Open in External Terminal" button
- Kekang spawn native terminal dengan `claude --resume <session-id>` — same session continues in TUI
- Return to Kekang chat when done, session state synced

**Trade-offs (disclosed upfront):**

1. **Checkpoint/rewind not supported in Kekang chat** (as of 2026) — use external terminal for `/undo` workflows
2. **Some TUI-only slash commands may not render perfectly** — fallback to terminal for edge cases
3. **Control protocol is internal/undocumented** — insulated behind `ClaudeCliService`; feature-detect at startup; fallback plan = Node sidecar (see Risk table)

---

## 5. Non-Goals

**Kekang TIDAK lakukan:**

- ❌ Pengganti Claude Code engine (Kekang wraps `claude` CLI, engine tetap Anthropic's)
- ❌ Custom LLM inference atau routing
- ❌ Reimplement tool loop (CLI handles it)
- ❌ Cloud sync (semua local-first)
- ❌ Multi-user
- ❌ Support stack di luar 6 Rozzy pake + Rust/Tauri (buat self-dogfood Kekang) — skip Go, Java, Ruby, dll
- ❌ Orchestrator paralel agent runtime (Orkes/Conductor teritori)
- ❌ Token/traffic routing (skip 9Router)
- ❌ Proxy compression (delegate ke RTK)
- ❌ Output compression (delegate ke Caveman/Ponytail)
- ❌ Cost tracking granular per provider (basic only)
- ❌ "Yolo mode" enabler — dengan sengaja refuse skip harness phases
- ❌ Direct Anthropic API integration (all via Claude Code subscription)
- ❌ Distribusi komersial (personal use only, tidak untuk dijual)
- ❌ **Ship Python atau Node runtime** — backend murni Rust, frontend static export

**Klarifikasi tentang chat interface:** Kekang punya chat UI tapi ini BUKAN violation "not replacing Claude Code" — chat UI cuma render surface untuk proses `claude` CLI. Engine, skills, subagents, MCP, semua Anthropic's. Kalau CLI protocol berubah, Kekang mengikuti; kalau external terminal experience needed, tetap tersedia via fallback button.

---

## 6. Technical Architecture

### 6.1 Tech Stack

| Layer | Choice | Rationale |
|-------|--------|-----------|
| App shell | Tauri 2 | Native, ringan, webview UI + Rust backend, distribusi binary kecil |
| Language (backend) | Rust (stable ≥ 1.80) | Native perf, memory safety, process/DB/FS control |
| **Claude integration** | **`claude` CLI over stream-json control protocol** | **No SDK runtime di-ship, full control, subscription auth inherit** |
| Language (frontend) | TypeScript | Type-safe UI |
| UI framework | Next.js 15 (App Router, `output: 'export'`) | Modern React, familiar, static export cocok Tauri |
| UI kit | Tailwind v4 + shadcn/ui (Radix) | Fast, composable, dark-first |
| State | TanStack Query + Zustand | Server(ish) state + UI state |
| Bindings | `tauri-specta` + `specta` | Type-safe Rust→TS command/event types |
| Storage | SQLite via `sqlx` (sqlite, runtime-tokio, migrate) | Zero-config, single-user OK, async |
| File watching | `notify` | Cross-platform, mature |
| Async runtime | `tokio` | Process, timers, channels, tasks |
| Config | TOML via `toml` + `serde` | Human-readable |
| Notifications | `tauri-plugin-notification` | Cross-platform OS notif |
| Process mgmt | `tokio::process` | Spawn `claude`, tools, GC, AGENTS.md gen |
| Templating | Rust `rust-embed` + `handlebars` | Bootstrap templates, prompt templates |
| Markdown render | `react-markdown` + `shiki` | Chat message rendering + code highlight |
| Virtual list | `@tanstack/react-virtual` | Long chat transcripts |
| Scheduler | `tokio` interval + persisted `next_run_at` | GC schedule |
| Logging | `tracing` + `tracing-subscriber` | Structured logs ke `~/.kekang/logs/` |
| Packaging | Tauri bundler (`.dmg`, `.AppImage`, `.msi`) | Native installers |

**Not needed:** Direct `anthropic` SDK, LangChain, LiteLLM, `claude-agent-sdk` (Python/TS), Python, Node server runtime. The `claude` CLI covers all LLM interaction needs.

### 6.2 Claude CLI Integration

**Rust `claude/` module owns one long-lived child process per chat session.**

**Process spawn:**
```
claude -p \
  --input-format stream-json \
  --output-format stream-json \
  --verbose \
  --permission-mode default \
  --session-id <uuid> \
  --model <alias|id> \
  --max-budget-usd <n> \
  --mcp-config <project>/.mcp.json \
  --settings <project>/.claude/settings.json
env: CLAUDE_CONFIG_DIR=<profile path>
cwd: <project path>
```

- Resume pakai `--resume <uuid>` (bukan `--session-id`).
- Fork pakai `--fork-session`.
- Model/effort di-set saat spawn, bisa diganti mid-session via control request.

**Runtime shape per session (tokio):**
- **writer task** — `mpsc<ClientMsg>` → serialize newline-delimited JSON → child stdin. User prompt, control_response, interrupt.
- **reader task** — child stdout `BufReader::lines()` → parse JSON → classify → persist + emit Tauri event.
- session registry di Tauri `State` (`Mutex<HashMap<SessionId, SessionHandle>>`).

**Message types handled (stdout):**

| type | aksi |
|------|------|
| `system` (init) | simpan session meta, tool list, model |
| `assistant` | append `chat_messages`, emit message |
| `user` (tool_result) | append tool result |
| `stream_event` | partial delta → emit `chat://delta` |
| `result` | turn selesai, cost/token → `feature_metrics` |
| `control_request` (subtype `can_use_tool`) | **approval gate** → emit `chat://permission` |
| `control_response` | ack |

**Approval flow:**
```
CLI --control_request(can_use_tool)--> Rust --emit chat://permission--> Next UI
Next UI --invoke chat_respond_permission{sessionId,requestId,behavior}--> Rust
Rust --control_response--> CLI stdin
```
- Handshake `initialize` control_request dari CLI di-handle otomatis saat spawn (bukan ke UI).
- Timeout 30s → auto-deny dengan alasan jelas.
- "Approve Always" → catat `chat_auto_approvals`, jawab allow untuk pola itu, gak emit lagi.
- "Deny" → kirim control_response deny dengan reason ke CLI.

**Tauri events (typed, per session):**
`chat://delta`, `chat://message`, `chat://tool`, `chat://permission`, `chat://result`, `chat://error`, `chat://closed`.

**Interrupt:** kirim control_request interrupt; fallback kirim SIGINT ke child.

**Isolasi:** seluruh detail protocol di `claude/protocol.rs` + `claude/session.rs`. Command layer cuma expose `chat_send`, `chat_respond_permission`, `chat_interrupt`, `chat_new_session`, `chat_resume`, `chat_fork`, `chat_export`.

**Feature-detect:** startup jalankan `claude --version` → simpan di `app_meta`; tolak kalau < min version; probe kontrol yang dibutuhkan. Kalau tidak cocok → degrade ke external terminal + pesan jelas.

**Fallback plan:** kalau protocol raw terbukti rapuh di maintenance jangka panjang, ganti implementasi `ClaudeCliService` ke Node sidecar (`@anthropic-ai/claude-agent-sdk`) tanpa ubah command layer (approach C).

### 6.3 Storage & Data Ownership

**SQLite tetap, dimiliki Rust.** `~/.kekang/kekang.db`, akses eksklusif via repository layer. Frontend gak pernah sentuh DB langsung.

Pool config: WAL, `foreign_keys=ON`, `busy_timeout=5000`, `max_connections=5`.

Schema: seluruh tabel v1.1 (Section 7) + perubahan additive:
- `app_meta(key, value)` — `cli_version`, `schema_version`, `first_run_at`.
- `chat_sessions.cli_version TEXT` — versi CLI saat session dibuat.

Repositories (satu per aggregate): `projects`, `profiles`, `phases`, `bootstrap`, `features`, `tasks`, `tools`, `presets`, `skills`, `graphify`, `mcp`, `gc`, `metrics`, `retros`, `chat`. Command layer panggil repo; gak ada raw SQL di command.

**Filesystem ownership:**
- Backup: `write temp → fsync → rename` (atomic), ke `~/.kekang/backups/<project>/<timestamp>/`.
- Config: `~/.kekang/config.toml` (global) + `<project>/.kekang/config.toml`.
- Session export: `~/.kekang/sessions/exports/`.
- Semua file op lewat `storage/fs.rs` helper (atomic + backup hook).

### 6.4 Frontend (Next.js in Tauri webview)

**Build mode:** `output: 'export'` → static HTML/JS di `apps/desktop/out`, dimuat Tauri sebagai `frontendDist`. Dev: `next dev` via `devUrl http://localhost:3000`. Tidak ada Node server di produksi.

**Routing (static-export safe):** no dynamic `[id]`. Top-level route statis per bagian + entity via query params:
```
/                    dashboard
/project?id=…        project detail (5-phase view + chat)
/feature?id=…&fid=…
/settings            profiles, tools, presets
/inbox               escalations
/metrics
```
`useSearchParams` + Suspense boundary.

**State:**
| Jenis | Tool |
|-------|------|
| Data dari Rust | TanStack Query; invalidate saat Tauri event masuk |
| UI state (active project, panel mode) | Zustand |
| Chat realtime | listener `chat://*` → update Zustand store + virtualized list |

**Tauri bridge:** `@tauri-apps/api` (`invoke`, `listen`) dibungkus di `lib/tauri/`. Type-safe generated bindings via `tauri-specta` → `packages/bindings` (Rust command + event enum → TS discriminated union). Gak ada `invoke("string")` manual.

**UI kit:** Tailwind v4 + shadcn/ui (Radix), `lucide-react`, `motion` (chat transitions), `@tanstack/react-virtual` (message list), `react-markdown` + `shiki`, `cmdk` (slash-command autocomplete + command palette), `react-hook-form` + `zod` (forms).

**Layout:** sidebar nav + main region; chat panel modes `docked` (grid 60/40) / `fullscreen` / `minimized` via Zustand. Streaming delta di-batch pakai `requestAnimationFrame` biar render lancar walau token deras.

### 6.5 Component Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Kekang App (Tauri 2)                      │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │        Frontend — Next.js (static export, webview)      │ │
│  │  Dashboard · ProjectDetail (5-phase) · ChatPanel ·      │ │
│  │  BootstrapWizard · AgentsGenerator · SubagentEditor ·   │ │
│  │  ToolsManager · SkillManager · ProfileSettings · Inbox ·│ │
│  │  GraphifyManager · GCScheduler · MetricsDashboard ·     │ │
│  │  RetroCapture · PresetManager · SessionManager          │ │
│  │                                                            │
│  │  lib/tauri/{commands,events}.ts  (tauri-specta bindings) │
│  └───────────────────────┬────────────────────────────────┘ │
│                          │ invoke / listen (Tauri IPC)        │
│  ┌───────────────────────▼────────────────────────────────┐ │
│  │                 commands/ (thin API surface)             │ │
│  │  #[tauri::command] wrappers → domain/harness/services    │ │
│  └───────────────────────┬────────────────────────────────┘ │
│  ┌───────────────────────▼────────────────────────────────┐ │
│  │         Harness Orchestration (core value, Rust)         │ │
│  │  PhaseTracker · WorkflowEnforcer · StateValidator ·      │ │
│  │  BootstrapService · StackScaffolds                       │ │
│  └───────────────────────┬────────────────────────────────┘ │
│  ┌───────────────────────▼────────────────────────────────┐ │
│  │                  Services (Rust)                         │ │
│  │  ClaudeCliService · ChatSessionService · PermissionBridge│ │
│  │  ProjectManager · ProfileManager · Watcher · Notifier ·  │ │
│  │  SubagentService · SkillService · ToolInstaller ·        │ │
│  │  GraphifyService · GcScheduler · GcRunner ·              │ │
│  │  MetricsCollector · RetroService · TerminalLauncher      │ │
│  │  tools/adapters/{rtk,caveman,ponytail,graphify,context7,…}│ │
│  └───────────────────────┬────────────────────────────────┘ │
│  ┌───────────────────────▼────────────────────────────────┐ │
│  │             Storage & Infrastructure (Rust)              │ │
│  │  sqlx · repositories · fs (atomic+backup) · notify ·     │ │
│  │  tokio::process · Tauri events · tauri-plugin-*          │ │
│  └───────────────────────┬────────────────────────────────┘ │
└──────────────────────────┼───────────────────────────────────┘
            │              │                    │
            ▼              ▼                    ▼
   ┌──────────────┐  ┌──────────────┐   ┌──────────────────┐
   │  ~/.kekang/  │  │ Target repos │   │  claude CLI      │
   │  db/config/  │  │ + Global     │   │  (long-lived      │
   │  logs/       │  │ AGENTS.md,   │   │   per session)    │
   │  backups/    │  │ docs/,       │   └──────────────────┘
   │  sessions/   │  │ .claude/,    │
   │  templates/  │  │ .agent-      │
   └──────────────┘  │ workspace/,  │
                     │ graphify-out/│
                     └──────────────┘
```

**Key components (v2.0):**

- **ClaudeCliService** — spawn/lifecycle proses `claude`, writer/reader task, framing stream-json, dispatch event ke UI.
- **PermissionBridge** — terima `control_request(can_use_tool)`, emit ke UI, tunggu keputusan (timeout 30s), balas `control_response`. Auto-approve policy dari `WorkflowEnforcer`.
- **ChatSessionService** — persist/resume/fork session; interface `chat_sessions` + `chat_messages`.
- **Watcher** — `notify` recursive, debounce, emit event.
- **GcScheduler** — tokio timer + persisted schedule.
- **TerminalLauncher** — fallback spawn `claude --resume <uuid>` di terminal native.

**Layer clarity:**
- `ClaudeCliService` = Service (plumbing), bukan orchestration.
- Approval *policy* (tool mana auto-approve) = Harness (`WorkflowEnforcer`) — bagian disiplin.
- Approve/deny *UI* = Frontend (`ChatPanel`).

**Layer rules:**
- Frontend presentational only — no DB/FS access langsung, semua via `invoke`.
- Rust single writer untuk SQLite + filesystem + process.
- Harness layer gak boleh manggil Tauri API langsung (biar testable tanpa runtime).

### 6.6 Cross-Cutting Concerns

- **Errors:** commands return `Result<T, AppError>` (`thiserror` → serde tagged enum). Frontend map ke toast. Error reader-task → `chat://error`, bukan crash. Panics isolated per task.
- **Logging:** `tracing` → `~/.kekang/logs/kekang.log` (rotasi), level dari config.
- **Security:** no secret disimpan; auth cuma `CLAUDE_CONFIG_DIR`; `bypassPermissions` butuh konfirmasi + banner. Tool install: tampilkan command dulu, auto-run cuma pola aman tanpa sudo. Semua path user di-canonicalize + jail ke project root. Never log token/credential.
- **Config:** global + per-project TOML dengan `schema_version`.
- **Observability:** structured logs untuk command entry/exit, session lifecycle, protocol frames (redacted).

### 6.7 Testing Strategy

- **Rust unit:** state machines, validators, JSONL framing/parse (pake fake transport), scaffold golden tests. `cargo test`.
- **Rust integration:** temp SQLite (sqlx + `tempfile`), repository CRUD, migration apply.
- **Protocol spike test (gate):** spawn real `claude`, prompt, `can_use_tool` allow + deny, `--resume`, assert partial stream. Tidak jalan di CI tanpa auth → `#[ignore]`, dijalankan manual.
- **Frontend:** Vitest + React Testing Library, mock `invoke`/`listen`.
- **E2E:** Playwright + `tauri-driver` (bootstrap flow, chat approve/deny).
- **Gates:** `cargo fmt --check`, `cargo clippy -D warnings`, `tsc --noEmit`, `eslint`, `prettier`.

### 6.8 UI Design System (Google Stitch)

Seluruh UI Kekang mengikuti design system yang dikelola di **Google Stitch**, bukan ditentukan ad-hoc di kode.

- **Stitch project:** "Kekang Developer Harness Dashboard" — `projects/13020582949250899293`
  (<https://stitch.withgoogle.com/projects/13020582949250899293>)
- **MCP server:** `stitch` (remote, di opencode — lihat `AGENTS.md`)
- **Local mirror:** `docs/DESIGN.md` (salinan design system + daftar 17 screen dari Stitch)
- **Tokens:** dark-only, primary accent `#7C3AED`, Inter (UI) + JetBrains Mono (code/telemetry), radius 6/8/12px, spacing 4px grid.

**Aturan (MANDATORY):**
1. Sebelum bangun screen/komponen, fetch screen Stitch terkait (`get_screen` / `list_screens`) dan cocokkan layout, token, density, states.
2. Pakai hanya token dari `docs/DESIGN.md`. No ad-hoc color/font/radius/spacing.
3. Kalau screen belum ada di Stitch, buat dulu di Stitch (`generate_screen_from_text` / `edit_screens`), lalu mirror token balik ke `docs/DESIGN.md`.
4. Kalau Stitch vs `docs/DESIGN.md` beda → Stitch menang, update mirror.
5. Semua screen di Stitch bertipe DESKTOP; layout utama 3-kolom (rail 260px · pane fluid · inspector 340–480px).

Tabel mapping screen Stitch lengkap ada di `docs/DESIGN.md`.

---

## 7. Data Model

Complete SQLite schema di `~/.kekang/kekang.db`:

```sql
-- App metadata (v2.0)
CREATE TABLE app_meta (
    key   TEXT PRIMARY KEY,
    value TEXT NOT NULL
);  -- cli_version, schema_version, first_run_at

-- Profile management
CREATE TABLE profiles (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL UNIQUE,
    display_name TEXT NOT NULL,
    config_dir TEXT NOT NULL,
    color TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Project registry
CREATE TABLE projects (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    path TEXT NOT NULL UNIQUE,
    stack TEXT,
    profile_id INTEGER REFERENCES profiles(id),
    active_preset_id INTEGER REFERENCES tool_presets(id),
    is_active BOOLEAN DEFAULT TRUE,
    last_opened_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE project_repos (
    id INTEGER PRIMARY KEY,
    project_id INTEGER REFERENCES projects(id) ON DELETE CASCADE,
    repo_path TEXT NOT NULL,
    repo_label TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Phase tracking (5-phase workflow)
CREATE TABLE project_phases (
    id INTEGER PRIMARY KEY,
    project_id INTEGER REFERENCES projects(id) ON DELETE CASCADE,
    phase INTEGER NOT NULL,             -- 1-5
    status TEXT NOT NULL,               -- not_started, in_progress, complete, blocked
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    metadata_json TEXT,
    UNIQUE(project_id, phase)
);

-- Bootstrap detail (Phase 1)
CREATE TABLE bootstrap_checklist (
    id INTEGER PRIMARY KEY,
    project_id INTEGER REFERENCES projects(id) ON DELETE CASCADE,
    item TEXT NOT NULL,                 -- agents_md, docs, linter, hooks, subagents, workspace, graphify
    status TEXT NOT NULL,               -- pending, complete, skipped
    completed_at TIMESTAMP,
    UNIQUE(project_id, item)
);

-- Feature workflow (Phase 2)
CREATE TABLE features (
    id INTEGER PRIMARY KEY,
    project_id INTEGER REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    spec_file TEXT,
    plan_file TEXT,
    state TEXT NOT NULL,                -- brainstorming, planning, executing, reviewing, merged, abandoned
    started_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP,
    merged_pr_url TEXT
);

CREATE TABLE tasks (
    id INTEGER PRIMARY KEY,
    feature_id INTEGER REFERENCES features(id) ON DELETE CASCADE,
    task_id TEXT NOT NULL,
    title TEXT NOT NULL,
    state TEXT NOT NULL,                -- pending, in_progress, done, escalated
    retry_count INTEGER DEFAULT 0,
    escalated_at TIMESTAMP,
    completed_at TIMESTAMP,
    UNIQUE(feature_id, task_id)
);

-- AGENTS.md generation history
CREATE TABLE agents_generation_history (
    id INTEGER PRIMARY KEY,
    project_id INTEGER REFERENCES projects(id),
    prompt TEXT,
    generated_content TEXT,
    skill_invoked TEXT DEFAULT '/kekang:generate-agents',
    graphify_report_used BOOLEAN DEFAULT FALSE,
    subprocess_duration_ms INTEGER,
    applied BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Escalation tracking
CREATE TABLE escalations_seen (
    id INTEGER PRIMARY KEY,
    project_id INTEGER REFERENCES projects(id),
    file_path TEXT NOT NULL,
    task_id TEXT,
    seen_at TIMESTAMP,
    resolved_at TIMESTAMP,
    UNIQUE(file_path)
);

-- Tools registry (RTK, Caveman, Ponytail, Graphify, Superpowers, dll)
CREATE TABLE tools (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL UNIQUE,
    type TEXT NOT NULL,                 -- cli, skill, mcp
    tier INTEGER DEFAULT 2,             -- 1=recommended, 2=optional, 3=experimental
    version TEXT,
    installed BOOLEAN DEFAULT FALSE,
    install_method TEXT,                -- brew, npm, curl, uv, manual
    config_path TEXT,
    last_health_check TIMESTAMP,
    health_status TEXT,                 -- active, inactive, error
    metadata_json TEXT
);

-- Presets (tool + skill combinations)
CREATE TABLE tool_presets (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT,
    global_tools TEXT,                  -- CSV: 'rtk,superpowers,graphify'
    project_tools TEXT,                 -- JSON: {"caveman": "concise", "ponytail": "Lite"}
    custom_skills TEXT,
    profile_id INTEGER REFERENCES profiles(id), -- Preset scope
    is_default BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Per-project skill state
CREATE TABLE project_skills (
    id INTEGER PRIMARY KEY,
    project_id INTEGER REFERENCES projects(id) ON DELETE CASCADE,
    skill_name TEXT NOT NULL,
    enabled BOOLEAN DEFAULT TRUE,
    config_json TEXT,
    UNIQUE(project_id, skill_name)
);

-- Graphify integration
CREATE TABLE graphify_graphs (
    id INTEGER PRIMARY KEY,
    project_id INTEGER REFERENCES projects(id) ON DELETE CASCADE UNIQUE,
    graph_path TEXT NOT NULL,
    report_path TEXT NOT NULL,
    mcp_registered BOOLEAN DEFAULT FALSE,
    last_built_at TIMESTAMP,
    last_updated_at TIMESTAMP,
    node_count INTEGER,
    edge_count INTEGER,
    god_node_count INTEGER,
    languages_json TEXT
);

-- MCP servers per project
CREATE TABLE mcp_registrations (
    id INTEGER PRIMARY KEY,
    project_id INTEGER REFERENCES projects(id) ON DELETE CASCADE,
    server_name TEXT NOT NULL,          -- graphify, context7, dll
    config_path TEXT NOT NULL,
    active BOOLEAN DEFAULT TRUE,
    registered_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(project_id, server_name)
);

-- GC scheduling
CREATE TABLE gc_schedules (
    id INTEGER PRIMARY KEY,
    project_id INTEGER REFERENCES projects(id) ON DELETE CASCADE UNIQUE,
    cron_expression TEXT NOT NULL,      -- default: '0 9 * * MON'
    enabled BOOLEAN DEFAULT TRUE,
    last_run_at TIMESTAMP,
    next_run_at TIMESTAMP
);

CREATE TABLE gc_reports (
    id INTEGER PRIMARY KEY,
    project_id INTEGER REFERENCES projects(id) ON DELETE CASCADE,
    run_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    report_file TEXT NOT NULL,
    auto_fix_count INTEGER DEFAULT 0,
    needs_review_count INTEGER DEFAULT 0,
    needs_decision_count INTEGER DEFAULT 0
);

-- Observability
CREATE TABLE feature_metrics (
    id INTEGER PRIMARY KEY,
    feature_id INTEGER REFERENCES features(id),
    total_tokens INTEGER,
    total_cost_estimate REAL,
    retry_rate REAL,
    escalation_count INTEGER,
    spec_drift_count INTEGER,
    time_to_merge_minutes INTEGER,
    collected_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE tool_metrics (
    id INTEGER PRIMARY KEY,
    tool_name TEXT NOT NULL,
    project_id INTEGER REFERENCES projects(id),
    date DATE NOT NULL,
    tokens_saved INTEGER,
    commands_processed INTEGER,
    raw_json TEXT,
    collected_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tool_name, project_id, date)
);

CREATE TABLE retros (
    id INTEGER PRIMARY KEY,
    feature_id INTEGER REFERENCES features(id),
    what_worked TEXT,
    what_didnt TEXT,
    new_skill_suggestion TEXT,
    new_guardrail_suggestion TEXT,
    captured_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Chat sessions (CLI integration)
CREATE TABLE chat_sessions (
    id INTEGER PRIMARY KEY,
    project_id INTEGER REFERENCES projects(id) ON DELETE CASCADE,
    session_uuid TEXT NOT NULL UNIQUE,  -- CLI session ID untuk resume
    title TEXT,                          -- auto-generated dari first message atau user-set
    model TEXT,                          -- 'sonnet', 'opus', 'haiku', 'inherit'
    permission_mode TEXT DEFAULT 'default',
    max_budget_usd REAL DEFAULT 5.0,
    max_turns INTEGER,
    profile_id INTEGER REFERENCES profiles(id),
    feature_id INTEGER REFERENCES features(id),  -- linked feature (optional)
    parent_session_id INTEGER REFERENCES chat_sessions(id),  -- for forks
    state TEXT DEFAULT 'active',         -- active, paused, completed, forked
    cli_version TEXT,                    -- versi CLI saat session dibuat (v2.0)
    total_tokens INTEGER DEFAULT 0,
    total_cost_usd REAL DEFAULT 0.0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_activity_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE chat_messages (
    id INTEGER PRIMARY KEY,
    session_id INTEGER REFERENCES chat_sessions(id) ON DELETE CASCADE,
    role TEXT NOT NULL,                  -- user, assistant, tool_use, tool_result, system
    content TEXT,                        -- text content or JSON for structured
    tool_name TEXT,                      -- for tool_use messages
    tool_input_json TEXT,                -- tool arguments
    tool_output_json TEXT,               -- tool result
    approval_state TEXT,                 -- pending, approved, denied, auto_approved, null (n/a)
    approval_decided_at TIMESTAMP,
    tokens_used INTEGER,
    cost_usd REAL,
    subagent_role TEXT,                  -- planner, executor, reviewer, gc-agent, null
    sequence INTEGER NOT NULL,           -- message order in session
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(session_id, sequence)
);

CREATE TABLE chat_auto_approvals (
    id INTEGER PRIMARY KEY,
    session_id INTEGER REFERENCES chat_sessions(id) ON DELETE CASCADE,
    tool_pattern TEXT NOT NULL,          -- e.g., 'Read', 'Bash:git *', 'Write'
    scope TEXT NOT NULL,                 -- 'session', 'project', 'global'
    expires_at TIMESTAMP,                -- null = never
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## 8. Directory Structure

**App code:**

```
kekang/
├── apps/
│   └── desktop/
│       ├── src/                          # Next.js UI
│       │   ├── app/                      # App Router (static routes)
│       │   │   ├── page.tsx              # dashboard
│       │   │   ├── project/page.tsx
│       │   │   ├── feature/page.tsx
│       │   │   ├── settings/page.tsx
│       │   │   ├── inbox/page.tsx
│       │   │   ├── metrics/page.tsx
│       │   │   └── layout.tsx
│       │   ├── components/               # shadcn + shared
│       │   ├── features/                 # feature UI modules
│       │   │   ├── dashboard/
│       │   │   ├── chat/                 # ChatPanel, ApprovalPrompt, MessageRenderer, SessionManager
│       │   │   ├── bootstrap/
│       │   │   ├── workflow/
│       │   │   ├── agents/
│       │   │   ├── subagents/
│       │   │   ├── tools/
│       │   │   ├── skills/
│       │   │   ├── profiles/
│       │   │   ├── inbox/
│       │   │   ├── graphify/
│       │   │   ├── gc/
│       │   │   ├── metrics/
│       │   │   ├── retro/
│       │   │   └── presets/
│       │   ├── lib/
│       │   │   ├── tauri/                # invoke + listen wrappers
│       │   │   └── utils/
│       │   └── stores/                   # zustand
│       ├── src-tauri/
│       │   ├── src/
│       │   │   ├── main.rs
│       │   │   ├── lib.rs
│       │   │   ├── commands/             # #[tauri::command] thin surface
│       │   │   │   ├── projects.rs
│       │   │   │   ├── profiles.rs
│       │   │   │   ├── phases.rs
│       │   │   │   ├── bootstrap.rs
│       │   │   │   ├── features.rs
│       │   │   │   ├── chat.rs
│       │   │   │   ├── tools.rs
│       │   │   │   ├── skills.rs
│       │   │   │   ├── graphify.rs
│       │   │   │   ├── gc.rs
│       │   │   │   ├── metrics.rs
│       │   │   │   └── presets.rs
│       │   │   ├── domain/               # entities + state machines
│       │   │   ├── harness/
│       │   │   │   ├── phase_tracker.rs
│       │   │   │   ├── workflow_enforcer.rs
│       │   │   │   ├── state_validator.rs
│       │   │   │   ├── bootstrap_service.rs
│       │   │   │   └── stack_scaffolds/
│       │   │   │       ├── mod.rs
│       │   │   │       ├── laravel.rs
│       │   │   │       ├── nestjs.rs
│       │   │   │       ├── nextjs.rs
│       │   │   │       ├── typescript.rs
│       │   │   │       ├── php.rs
│       │   │   │       └── python.rs
│       │   │   ├── services/
│       │   │   │   ├── project_manager.rs
│       │   │   │   ├── profile_manager.rs
│       │   │   │   ├── watcher.rs
│       │   │   │   ├── notifier.rs
│       │   │   │   ├── subagent_service.rs
│       │   │   │   ├── skill_service.rs
│       │   │   │   ├── tool_installer.rs
│       │   │   │   ├── graphify_service.rs
│       │   │   │   ├── gc_scheduler.rs
│       │   │   │   ├── gc_runner.rs
│       │   │   │   ├── metrics_collector.rs
│       │   │   │   ├── retro_service.rs
│       │   │   │   └── terminal_launcher.rs
│       │   │   ├── claude/
│       │   │   │   ├── mod.rs
│       │   │   │   ├── session.rs         # lifecycle + registry
│       │   │   │   ├── protocol.rs        # stream-json framing + control protocol
│       │   │   │   ├── messages.rs        # typed message enums
│       │   │   │   ├── permission_bridge.rs
│       │   │   │   └── session_store.rs
│       │   │   ├── tools/adapters/
│       │   │   │   ├── mod.rs             # ToolAdapter trait
│       │   │   │   ├── rtk.rs
│       │   │   │   ├── caveman.rs
│       │   │   │   ├── ponytail.rs
│       │   │   │   ├── graphify.rs
│       │   │   │   ├── context7.rs
│       │   │   │   ├── superpowers.rs
│       │   │   │   └── cavemem.rs
│       │   │   ├── storage/
│       │   │   │   ├── mod.rs             # pool + migrations
│       │   │   │   ├── fs.rs              # atomic write + backup
│       │   │   │   └── repositories/
│       │   │   ├── events.rs
│       │   │   ├── config.rs
│       │   │   └── error.rs
│       │   ├── migrations/
│       │   │   └── 0001_init.sql
│       │   ├── templates/                 # rust-embed: docs, linters, hooks, subagents
│       │   ├── icons/
│       │   ├── Cargo.toml
│       │   └── tauri.conf.json
│       ├── next.config.ts                # output: 'export'
│       ├── tailwind.config.ts
│       ├── components.json               # shadcn
│       └── package.json
├── packages/
│   ├── bindings/                          # tauri-specta generated TS
│   └── config/                            # shared tsconfig/eslint/prettier
├── docs/
│   ├── SPEC.md
│   └── Kekang-generate-agent-skill.md
├── package.json                           # pnpm workspace
└── README.md
```

**Kekang data:**

```
~/.kekang/
├── kekang.db                       # SQLite
├── config.toml
├── logs/
├── presets/
├── backups/                        # File backups sebelum overwrite
│   └── <project>/<timestamp>/
├── sessions/
│   ├── <session-uuid>.json         # session state cache
│   └── exports/                    # user-exported transcripts
└── templates/                      # User custom templates
```

**Target repo (after bootstrap):**

```
project/
├── AGENTS.md                       # Generated by /kekang:generate-agents
├── .kekang/
│   ├── config.toml
│   └── generate-request.md         # Ephemeral, deleted after use
├── docs/
│   ├── architecture/LAYERS.md
│   ├── golden-principles/
│   ├── decisions/
│   ├── api/
│   └── runbooks/
├── .claude/
│   ├── agents/                     # Subagent definitions
│   │   ├── planner.md
│   │   ├── executor.md
│   │   ├── spec-reviewer.md
│   │   ├── code-reviewer.md
│   │   └── gc-agent.md
│   └── settings.json               # Skill toggles
├── .mcp.json                       # MCP servers (graphify, dll)
├── .agent-workspace/               # Gitignored
│   ├── README.md
│   ├── current-spec.md
│   ├── current-plan.md
│   ├── task-queue.json
│   ├── task-log/
│   ├── review-reports/
│   ├── escalations/
│   └── gc-reports/
└── graphify-out/                   # Graphify data
    ├── graph.json
    ├── graph.html
    └── GRAPH_REPORT.md
```

---

## 9. Custom Skill: `/kekang:generate-agents`

Custom Claude Code skill yang di-install oleh Kekang (atau manual). Standalone-usable — works without Kekang. Full skill file di `docs/Kekang-generate-agent-skill.md`.

**Location after install:** `~/.claude/skills/kekang-generate-agents/SKILL.md`

**Invocation:**
- Via slash command: `/kekang:generate-agents`
- Via Kekang: spawn `claude` session dengan prompt `/kekang:generate-agents` dan `.kekang/generate-request.md` sudah disiapkan

**Behavior summary:**

1. **Context gathering** — read Kekang request file OR gather interactively
2. **Graphify integration** — read `graphify-out/GRAPH_REPORT.md` if exists (grounded architecture)
3. **Template selection** — pick from 7 stack templates (Laravel, NestJS, Next.js incl. static-export/Tauri variant, TypeScript, PHP, Python, Rust/Tauri) or combine for monorepo
4. **Generation** — output harness-compliant AGENTS.md dengan mandatory sections:
   - What This Is
   - Stack
   - Architecture Layers
   - Golden Principles
   - **Verification Requirements** (MANDATORY)
   - **Escalation Protocol** (MANDATORY)
   - **Reviewer Requirements** (MANDATORY)
   - **GC Compliance** (MANDATORY)
   - Where Things Live
   - What NOT to Do
   - Graphify Reality Check (kalau applicable)
5. **Quality checks** — max 120 lines, all sections present, concrete rules (not abstract), Graphify grounding if used
6. **Write & confirm** — atomic write to `AGENTS.md`, log to `.kekang/agents-generation-log.md`
7. **Cleanup** — delete `.kekang/generate-request.md`

**Key design:**
- Works standalone (skill only) or integrated (via Kekang UI)
- Improvement mode: preserves existing AGENTS.md custom content, adds missing harness sections
- Multi-stack aware: monorepo dengan Laravel + Next.js dapet root AGENTS.md + per-repo AGENTS.md
- Anti-patterns codified: refuses abstract principles ("write clean code"), demands concrete rules

---

## 10. Implementation Timeline

**Total:** 7 minggu MVP (Week 0 spike + 6 minggu fitur).

### Week 0 — Protocol Spike (GATE)

**Deliverable:**
- Throwaway script (boleh Node/bash, dibuang setelahnya) yang spawn `claude` dengan `--input-format stream-json --output-format stream-json --verbose`
- Verify: handshake `initialize`, streaming partial, round-trip `can_use_tool` allow + deny, `--resume <uuid>`, interrupt
- Catat format frame persis (control_request/control_response envelope)

**Success:** approve/deny tool call + resume + streaming terbukti lawan CLI real. **Kalau gagal → switch ke approach C (Node sidecar) sebelum Week 1.** Jangan invest di fondasi yang gak verified.

### Week 1 — Scaffold + Data Model + Bindings

**Deliverables:**
- pnpm workspace + Tauri 2 + Rust project layout
- CI (fmt, clippy, tsc, eslint)
- sqlx migrations = full schema v1.1 + `app_meta`
- Repositories + ProjectManager + ProfileManager
- PhaseTracker
- Next.js shell (dashboard skeleton, navigation, settings) + tauri-specta bindings

**Success:** add project + profile, phase state tracked di DB, UI shell jalan, command typed via bindings.

### Week 2 — Claude CliService + Chat UI (Core)

**Deliverables:**
- **ClaudeCliService** — process lifecycle, writer/reader task, framing
- **PermissionBridge** — `can_use_tool` → UI → control_response, timeout, auto-approve
- **ChatSessionService** — persist/resume/fork
- **ChatPanel UI** — message stream, streaming typewriter, input, slash autocomplete, session controls
- **ApprovalPrompt UI** — inline approve/deny/modify
- **SessionManager UI** — session list + switching
- Model selector, cost/token meter

**Success:** prompt Claude Code langsung dari Kekang, tool calls muncul dengan approval, session resume works, streaming lancar (benchmark render).

### Week 3 — Phase 1 Bootstrap + Static Layer

**Deliverables:**
- BootstrapService (7-step wizard: detect stack → Graphify → AGENTS.md → docs → linter → hooks → subagents → workspace)
- Stack scaffolds prioritas: **Laravel, NestJS, Next.js**
- Bootstrap Wizard UI dengan preview
- Templates embedded (docs, golden principles, linter, hooks, subagents)
- Atomic writes + backup + rollback
- TerminalLauncher (fallback native terminal dengan `CLAUDE_CONFIG_DIR`)

**Success:** bootstrap Laravel/NestJS/Next.js ke fully-harness-ready dalam 5 menit.

### Week 4 — Phase 2 Workflow + Custom Skill + Watcher

**Deliverables:**
- WorkflowEnforcer (validate spec, plan, task handoff format)
- Feature Workflow UI (brainstorming → planning → execution)
- **Custom skill `/kekang:generate-agents`** — build + install helper
- AGENTS.md generation via ClaudeCliService (bukan subprocess terpisah)
- Watcher service (`notify` di `.agent-workspace/` semua project)
- Notifier (`tauri-plugin-notification`)
- Escalations Inbox view

**Success:** feature workflow end-to-end via chat, notif escalation cross-project works, skill callable dari chat.

### Week 5 — Phase 3-5 (Verification + GC + Observability)

**Deliverables:**
- Verification loop tracking (retry, escalation counter)
- GC scheduler (tokio)
- GC runner (via ClaudeCliService dengan `/kekang:gc-scan`, leverage Graphify)
- GC report viewer + batch actions
- MetricsCollector (aggregate dari `chat_messages`, `task_log`, tool outputs)
- Metrics dashboard UI
- Retro capture UI + service

**Success:** full 5-phase operational, GC scheduled + actionable, metrics visible.

### Week 6 — Remaining Stacks + Supporting Features + Polish

**Deliverables:**
- Stack scaffolds sisanya: **TypeScript, PHP, Python**
- Tool Installer UI + adapters (RTK, Caveman, Ponytail, Graphify, Context7, Superpowers)
- Skill Manager UI (per-project toggle, presets)
- Subagent Editor UI (form-based)
- Preset save/load
- Graphify Manager UI
- Chat polish (auto-approve rules, keyboard shortcuts, syntax highlighting)
- External terminal fallback button
- Packaging (`.dmg` macOS first), README, docs

**Success:** MVP done, dipake harian mulai minggu 8.

### Realistic risk buffer

Kalau Week 2 (CLI integration + Chat UI) telat — cut lingkup Week 6 (skip TypeScript/PHP/Python scaffolds sampai post-MVP). Chat UI is critical; stack breadth is not.

---

## 11. Success Criteria

Personal tool, personal success metrics.

**Core (must-hit 7 dari 10):**

- [ ] Pake Kekang tiap hari selama 2 minggu setelah launch
- [ ] Minimal 2 project baru fully bootstrapped via Kekang (6+ checklist items)
- [ ] Minimal 3 feature end-to-end lewat 5-phase workflow (spec → plan → execute → review → merge)
- [ ] GC scan minimal 1x jalan otomatis, ada actionable findings
- [ ] Escalation notif minimal 1x save dari miss serious issue
- [ ] Zero data loss (semua file operation atomic, backup sebelum overwrite)
- [ ] Custom skill `/kekang:generate-agents` dipake minimal 3x, output diadopsi
- [ ] Graphify graph active di minimal 2 project, digunakan untuk AGENTS.md gen dan GC
- [ ] **Chat interface adalah primary interaction — 70%+ prompting via Kekang chat (bukan external terminal)**
- [ ] **Approval flow via UI works reliably — zero missed permission prompts**

**Nice-to-have (bonus):**

- [ ] Metric "cost per merged PR" tracked dan visible
- [ ] Retro capture dipake minimal 3x, ada suggestion actionable
- [ ] Preset apply ke project baru bikin bootstrap < 5 menit
- [ ] Skill toggle per project dipake minimal 5x (bukti feature dipake)
- [ ] **Installer size kecil, startup < 2s, idle memory wajar (tanpa runtime Python/Node)**

**Fail criteria:**

- < 5 dari 8 core items → post-mortem: apa yang salah? Feature over-designed? Timeline kegedean?
- Kekang jarang dibuka (< 3x/minggu) → Kekang gak solve real problem, pivot atau drop
- < 2 minggu retention → tool jelek atau habit gak berubah

---

## 12. Open Questions & Recommendations

Wajib dijawab sebelum start Week 1. Rekomendasi di kolom kanan — override kalau ada alasan kuat.

| # | Question | Recommendation |
|---|----------|----------------|
| 1 | Anthropic API key management | **N/A** — no API needed, all via `claude` CLI + `CLAUDE_CONFIG_DIR` |
| 2 | Multi-repo per project UI | **Tab dalam project detail**, bukan nested card |
| 3 | AGENTS.md generator model | **N/A** — via subscription |
| 4 | Notification filter | **Default all** + filter di Inbox view |
| 5 | Profile detection saat manual open | **Ignore** + warning kalau detect mismatch |
| 6 | Backup strategy overwrite files | `~/.kekang/backups/<project>/<timestamp>/` |
| 7 | Tool install permission | **Show command**, user execute (no sudo prompt) |
| 8 | Skill preset scope | **Per-profile** (Kantor preset beda dari Pribadi) |
| 9 | Ponytail mode default | **Lite** (balanced), Full opt-in, Ultra jarang |
| 10 | Phase enforcement strictness | **Soft warning default**, strict mode opt-in via settings |
| 11 | Bootstrap rollback strategy | **Atomic** — kalau fail di step X, rollback semua |
| 12 | GC frequency default | **Weekly** untuk active, **monthly** untuk maintenance |
| 13 | Metric collection method | **Parse Claude Code logs / `result` frames**, less invasive |
| 14 | Retro trigger | **Auto-prompt post-merge**, but skippable |
| 15 | Multi-stack repo (monorepo) | **Bootstrap per-repo** + root AGENTS.md yang reference each |
| 16 | Graphify graph freshness | **On-demand + weekly schedule + auto-refresh pre-GC** |
| 17 | Graphify deep mode | **Default incremental**, deep mode opt-in via UI button |
| 18 | Custom skill distribution | **Auto-install saat first project bootstrap**, dengan notice |
| 19 | Skill versioning | **Version check saat Kekang startup**, prompt update kalau ada |
| 20 | Graphify graph scope (multi-repo project) | **1 graph per repo**, aggregate view di Kekang UI |
| 21 | Min `claude` CLI version | **Detect at startup, gate implementation; bump saat protocol berubah** |
| 22 | Session cache lokasi | `~/.kekang/sessions/` (export + state cache), DB = source of truth |

---

## 13. Risks & Mitigation

| Risk | Impact | Mitigation |
|------|--------|-----------|
| **`claude` CLI control protocol berubah (internal/undocumented)** | **High** | **Isolate di `claude/protocol.rs`, feature-detect versi saat startup, pin min version, fallback plan: Node sidecar (approach C) tanpa ubah command layer** |
| **Protocol spike gagal (approve/deny/resume)** | **High** | **Week 0 gate — jangan lanjut sebelum terbukti; kalau gagal switch ke Node sidecar** |
| **`claude` CLI gak terinstall / versi terlalu lama** | **Medium** | **Startup check + onboarding guidance + disable chat dengan pesan jelas + external terminal tetap jalan kalau CLI ada** |
| Tauri webview streaming perf (token deras) | Medium | Benchmark Week 2, batch delta via rAF, virtualized list; worst case throttle |
| Next.js static export + dynamic routing | Medium | Query-param routing, no `[id]` segments (desain Section 6.4) |
| sqlx compile-time query friction | Low | Pakai `sqlx::query` runtime + `offline` mode untuk CI |
| Rust velocity (lebih lambat dari Python untuk UI plumbing) | Medium | Harness pure logic = highly testable; UI semua di TS; time-box weekly |
| macOS webview (WKWebView) rendering quirk | Low | Test early; fallback CSS; Tauri handle multi-platform |
| Tool CLI berubah format (RTK, Graphify, dll) | Medium | Version detection, adapter versioning, graceful degradation |
| Watcher CPU tinggi banyak project | Low | Debounce, limit watch depth |
| SQLite corruption saat crash | High | WAL mode, backup harian |
| Claude Code subagent format berubah | Medium | Version detection, warning kalau asing |
| Tool install fail (network, permission) | Low | Clear error + manual command fallback |
| **Scope creep (godaan add feature)** | **High** | **Strict: MVP 7 minggu, feature baru masuk backlog** |
| Skill config break `.claude/settings.json` | Medium | Backup sebelum edit, JSON schema validation |
| Bootstrap generates bad config | High | Preview mandatory, atomic write, easy rollback |
| GC subagent generates false positives | Medium | User review before action, category system |
| Metrics inaccurate | Low | "Estimated" label, explain source, allow correction |
| Phase enforcement terlalu strict | Medium | Soft default, strict opt-in |
| Graphify format changes | Medium | Version pin di adapter |
| Graph build slow (repo besar) | Low | Progress bar, background build, incremental default |
| Custom skill broken saat Claude Code update | Medium | Health check via startup, auto-repair prompt |
| MCP server registration conflict | Low | Detect existing entries, merge (bukan overwrite) |
| Graphify deep mode ate quota | Medium | UI warning + token estimate before trigger |
| User skip Graphify install | Low | Skill fallback "no-graph mode" dengan disclaimer |
| Subscription quota exhausted (5-hour limit) | High | Track call rate, warn if approaching limit |
| Session resume state corruption | Low | Export transcripts, start fresh referencing old transcript |
| can_use_tool callback deadlock | Medium | Timeout approval prompt (30s), fallback deny dengan alasan |

---

## 14. Prerequisites & Setup

### System prerequisites

```bash
# Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup component add clippy rustfmt

# Node + pnpm (build tooling only; no Node runtime shipped)
brew install node
corepack enable && corepack prepare pnpm@latest --activate

# Tauri CLI
cargo install tauri-cli --version "^2"

# Claude Code CLI (runtime dependency)
brew install --cask claude-code
claude --version   # catat versi, ini yang di-gate
```

### Pre-install tools (sebelum Kekang selesai)

```bash
# RTK
brew install rtk && rtk init -g

# Caveman (skill install)
curl -fsSL https://raw.githubusercontent.com/JuliusBrussee/caveman/main/install.sh | bash

# Ponytail — sesuai install docs

# Graphify (foundation)
uv tool install graphifyy
graphify install

# Superpowers plugin (via Claude Code marketplace atau manual)

# Install custom skill (bisa segera, tanpa tunggu Kekang)
mkdir -p ~/.claude/skills/kekang-generate-agents
cp docs/Kekang-generate-agent-skill.md ~/.claude/skills/kekang-generate-agents/SKILL.md
```

### Kekang Dev Environment

```bash
# Scaffold workspace
mkdir kekang && cd kekang
pnpm init
pnpm create tauri-app@latest apps/desktop --template react-ts   # lalu ganti frontend ke Next.js
cd apps/desktop
pnpm add next react react-dom
pnpm add -D typescript @types/react tailwindcss postcss autoprefixer
pnpm add @tanstack/react-query zustand @tauri-apps/api motion \
         @tanstack/react-virtual react-markdown shiki cmdk \
         react-hook-form zod lucide-react
pnpm dlx shadcn@latest init

# Rust deps (src-tauri/Cargo.toml)
cd src-tauri
cargo add tauri@2 tauri-plugin-notification tauri-plugin-opener \
          tauri-plugin-single-instance
cargo add tokio --features full
cargo add sqlx --features runtime-tokio,sqlite,migrate,chrono
cargo add serde serde_json toml thiserror anyhow chrono tracing tracing-subscriber
cargo add notify rust-embed handlebars specta tauri-specta
```

### Claude Code Profiles

```bash
# Setup dua profile (kalau belum)
mkdir -p ~/.claude-personal ~/.claude-work

# Configure Claude Code auth per profile
CLAUDE_CONFIG_DIR=~/.claude-personal claude auth login
CLAUDE_CONFIG_DIR=~/.claude-work claude auth login
```

### Protocol Spike (v2.0) — WAJIB sebelum Week 1

Throwaway script untuk verify stream-json control protocol. Contoh kerangka (Node, dibuang setelah spike):

```js
// spike.mjs — throwaway
import { spawn } from "node:child_process";

const p = spawn("claude", [
  "-p",
  "--input-format", "stream-json",
  "--output-format", "stream-json",
  "--verbose",
  "--permission-mode", "default",
  "--session-id", crypto.randomUUID(),
], {
  cwd: "/tmp",
  env: { ...process.env, CLAUDE_CONFIG_DIR: process.env.HOME + "/.claude-personal" },
  stdio: ["pipe", "pipe", "inherit"],
});

function send(obj) { p.stdin.write(JSON.stringify(obj) + "\n"); }

let buf = "";
p.stdout.on("data", (chunk) => {
  buf += chunk.toString();
  let i;
  while ((i = buf.indexOf("\n")) >= 0) {
    const line = buf.slice(0, i); buf = buf.slice(i + 1);
    if (!line.trim()) continue;
    const msg = JSON.parse(line);
    console.log("MSG:", msg.type, msg.subtype ?? "");

    // Handshake
    if (msg.type === "control_request" && msg.request?.subtype === "initialize") {
      send({ type: "control_response",
        response: { request_id: msg.request_id, subtype: "success", response: {} } });
    }
    // Approval gate
    if (msg.type === "control_request" && msg.request?.subtype === "can_use_tool") {
      console.log("PERMISSION ASK:", msg.request.tool_name, msg.request.input);
      send({ type: "control_response",
        response: {
          request_id: msg.request_id,
          subtype: "success",
          response: { behavior: "allow", updatedInput: msg.request.input },
        } });
    }
  }
});

send({ type: "user", message: { role: "user",
  content: [{ type: "text", text: "Create a file /tmp/kekang-spike.txt containing 'ok'. Then say done." }] } });
```

Jalankan: `node spike.mjs`

**Expected:** muncul `PERMISSION ASK: Write …`, tool jalan, stream assistant text, selesai.
**Kalau fail:**
- Auth error → `claude auth login` untuk profile itu
- No `control_request` → cek versi CLI, mungkin framing beda → dokumentasikan
- Timeout → network / CLI path issue

Kalau spike pass, foundation solid untuk Week 2.

---

## 15. Post-MVP Roadmap

Setelah MVP validated (2 minggu dogfood, 6+ dari 10 success criteria hit), consider add fitur berikut secara urutan value/effort:

**Priority 1 (High value, low-medium effort):**
1. **CLI wrapper** (3-5 hari) — power user commands: `kekang open <name>`, `kekang status`, `kekang gen-agents`
2. **Cross-project pattern detection** (1 minggu) — analyze retros across projects, suggest global golden principles
3. **Custom skill generator** (1 minggu) — prompt → skill file (mirip AGENTS.md gen tapi buat skill)

**Priority 2 (Medium value, medium effort):**
4. **Preset marketplace** (1-2 minggu) — share preset via URL/git repo
5. **Skill dependencies** (1 minggu) — skill A butuh skill B, auto-resolve
6. **Advanced retro analytics** (2-3 minggu) — trend analysis, correlation retry rate vs model choice
7. **Multi-machine sync** (1 minggu) — kalau punya 2 mesin
8. **Auto-updater** (3-5 hari) — `tauri-plugin-updater` + signed releases

**Priority 3 (Speculative, hold):**
9. **9Router integration mode** (1-2 minggu) — kalau butuh gateway routing
10. **Openclaw/Hermes hook** — timing depend on maturity
11. **Team mode** — kalau ekspansi jadi consulting product (violate current non-goal)

---

## Filosofi & Prinsip Guide

**Discipline saat build:**

1. **Dogfood dari Week 2.** Pake versi setengah-jadi buat kerjaan real. Kalau gak bisa dipake, itu sinyal scope salah.
2. **Log semua friction** di `~/.kekang/dogfood-notes.md`. Raw material buat iterate.
3. **Ship jelek dulu, polish nanti.** UI cukup functional. Cantik = week 6+.
4. **Resist feature request dari diri sendiri.** Kepikiran feature baru → tulis di `backlog.md`, jangan langsung code.
5. **Time-box weekly.** Kalau week ke-N gak selesai → cut scope, jangan extend timeline.

**Yang paling worth di-invest waktu upfront:**

- **Protocol isolation** (`claude/protocol.rs`) — CLI drift adalah risiko terbesar; boundary ini yang bikin Kekang tahan update
- **Harness orchestration layer** clean separation dari services → extensible + testable tanpa runtime Tauri
- **Stack scaffold templates** untuk 3 stack prioritas → foundation buat 3 stack sisanya
- **State validators** — validate handoff files, spec format, plan format
- **Atomic file operations** — data loss = kehilangan trust
- **Custom skill quality** — skill yang bagus reusable seumur hidup

**Yang boleh murahan dulu:**

- UI styling polish (fungsional dulu; shadcn default OK)
- Error messages (generic OK)
- Documentation (README singkat OK)

**Prioritas debugging (Week 2+):**

Trade-off? **Reliability > feature richness.** Data loss atau miss notif = kehilangan trust. Feature gap = user friction tapi recoverable.

**Prioritas Week 4:**

Kalau ada trade-off: **Bootstrap Wizard > Metrics Dashboard.** Bootstrap = immediate core value. Metrics = nice-to-have.

---

## Kesimpulan Filosofi

**Kekang bukan tool untuk shortcut discipline.** Kekang tool untuk **make discipline sustainable at scale**.

Manual discipline works untuk 1-2 project. Skala ke 5+ project multi-profile multi-session, discipline manual jadi mahal — mental load tinggi, konsistensi drop, drift accumulate. Kekang automate mechanical parts of discipline (bootstrap, validation, GC scheduling, metrics collection), leave decision parts to human (approval gates, escalation handling, merge decision, judgment calls).

**Yang Kekang jaga:**
- Setiap project fully bootstrapped (Phase 1 complete)
- Setiap feature lewat 5-phase workflow (no yolo mode)
- Handoff files valid dan meaningful
- GC jalan periodic
- Metric visible untuk retro

**Yang Rozzy tetap decide:**
- Spec quality (Kekang validate structure, gak validate content)
- Approval gates di setiap phase transition
- Escalation resolution
- Merge decision
- Pattern extraction dari retro

Ini bagi tugas yang sehat: **Kekang = mechanical enforcement + orchestration**, **Rozzy = judgment + creativity**.

**Grounded di real code understanding (Graphify), enforced via 5-phase workflow, powered by Claude Code subscription. No API key, no cloud, no bullshit. Native binary, no Python/Node runtime shipped.**

---

## Companion File

`docs/Kekang-generate-agent-skill.md` — Custom Claude Code skill yang bisa install standalone (tanpa Kekang selesai) dan langsung dipake untuk generate AGENTS.md per project. Semua template stack detail (Laravel, NestJS, Next.js incl. static-export/Tauri variant, TypeScript, PHP, Python, Rust/Tauri) di sana.

---

**End of specification — v2.0**

Ready for implementation. Next steps:
1. Jawab open questions Section 12 (rekomendasi sudah ada, override kalau perlu)
2. Setup dev environment (Section 14)
3. **Jalankan protocol spike (Section 14) — WAJIB verify sebelum Week 1**
4. Install baseline tools untuk dogfooding
5. Install custom skill immediately (tanpa tunggu Kekang)
6. Start Week 0 spike, lanjut Week 1
7. Ship MVP dalam 7 minggu

## Changelog v1.1 → v2.0

**Changed (stack swap):**
- Python 3.11 + Flet → **Rust + Tauri 2 + Next.js 15**
- `claude-agent-sdk` (Python) → **direct `claude` CLI over stream-json control protocol**
- `aiosqlite` → `sqlx`; `watchdog` → `notify`; `apscheduler` → `tokio`; `plyer` → `tauri-plugin-notification`
- `subprocess` → `tokio::process`
- New bindings layer: `tauri-specta`/`specta`
- New frontend stack: Tailwind + shadcn/ui + TanStack Query + Zustand

**Added:**
- Week 0 protocol spike (gate) + Node spike script (throwaway)
- `claude/` module (session, protocol, messages, permission_bridge)
- `app_meta` table + `chat_sessions.cli_version`
- CLI version detection + min-version gate
- Approach C (Node sidecar) documented as fallback
- Non-goal: no shipped Python/Node runtime
- Testing strategy section, cross-cutting concerns section

**Unchanged:**
- 5-phase harness workflow (still the core)
- All supporting features A–K (including integrated chat semantics)
- Graphify grounding
- Custom skill for AGENTS.md gen
- Multi-project, multi-profile, multi-repo support
- Subscription-only auth
- Data model (except additive columns)
- Open questions (plus 2 new)

**Removed:**
- Python-specific SDK risks → replaced by CLI-protocol risks
- "skip Rust" non-goal (now Rust is the backend)
