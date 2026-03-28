# Seeds — Comprehensive Repository Analysis

**Repository:** `jayminwest/seeds`  
**npm package:** `@os-eco/seeds-cli` v0.2.5  
**License:** MIT  
**Analyzed:** June 2025

---

## 1. What the Project Is and Does

### Purpose

**Seeds** is a git-native issue tracker designed specifically for AI agent workflows. It replaces [beads](https://github.com/steveyegge/beads) in the `overstory`/`mulch` ecosystem. The core thesis is radical simplicity: **the JSONL file IS the database** — no binary files, no daemon, no export pipeline, no sync step.

### Goals

1. **JSONL as the single source of truth** — plain text, diffable, git-mergeable
2. **Zero friction for multi-agent concurrent writes** — advisory file locks + atomic writes
3. **Git-native by design** — `merge=union` gitattribute, git is the audit trail
4. **AI-agent first** — structured `--json` output on every command, `sd prime` for context injection, `sd onboard` for auto-configuring CLAUDE.md/AGENTS.md
5. **Minimal footprint** — only two runtime dependencies (chalk + commander), runs directly on Bun with no build step

### Why It Exists (vs. Beads)

| Problem | Beads | Seeds |
|---------|-------|-------|
| Storage | 2.8 MB binary `beads.db` (can't diff/merge) | JSONL (diffable, mergeable) |
| Sync | 286 export-state tracking files | No sync — file IS the DB |
| Concurrency | `beads.db` lock contention | Advisory locks + atomic writes |
| Dependencies | Dolt embedded | chalk + commander |

### Target Audience

- **AI coding agents** (Claude Code, Overstory workers) that need a lightweight, concurrent-safe task tracker
- **Developers** working in multi-agent environments (git worktrees, parallel agents)
- **os-eco ecosystem** users (overstory, mulch, canopy toolchain)
- Solo developers who want a git-native CLI issue tracker with zero infra

---

## 2. Tech Stack and Key Dependencies

### Runtime & Language

| Concern | Choice | Notes |
|---------|--------|-------|
| **Runtime** | Bun ≥ 1.0.0 | Runs TypeScript directly, no compilation step |
| **Language** | TypeScript (strict) | `noUncheckedIndexedAccess`, no `any` allowed |
| **Module system** | ESM (`"type": "module"`) | Native ES modules throughout |

### Runtime Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `chalk` | `^5.6.2` | Terminal color output with os-eco forest palette (brand/accent/muted) |
| `commander` | `^14.0.3` | CLI argument/option parsing, help generation, subcommands |

### Dev Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `@biomejs/biome` | `^2.4.6` | Combined linter + formatter (replaces ESLint + Prettier) |
| `@types/bun` | latest | TypeScript types for Bun APIs |
| `typescript` | `^5.9.0` | Type checking (`tsc --noEmit`) |

### Built-in Bun/Node APIs Used (Zero Extra Deps)

- `Bun.file` / `Bun.write` — file reads and writes
- `node:fs` — advisory file locking (`openSync` with `wx` flag), atomic rename
- `node:crypto` — `randomBytes` for ID generation
- `node:path` — path joining

### Storage & Config

- **JSONL** — append-only, one JSON object per line, mutates via atomic temp-file rename
- **YAML** — minimal built-in parser (~27 LOC, `src/yaml.ts`), handles only flat key-value format; no external YAML dependency

### Tooling

- **Test runner:** `bun test` (Jest-compatible API, built-in)
- **Linter/Formatter:** Biome (tabs, 100-char line width)
- **Version management:** custom `scripts/version-bump.ts` keeping `package.json` + `src/index.ts` in sync
- **CI:** GitHub Actions with `oven-sh/setup-bun@v2`
- **Publishing:** npm (`@os-eco/seeds-cli`), published via `publish.yml` on version change detection

---

## 3. Project Structure Overview

```
seeds/
├── package.json                  # Bin: "sd" → ./src/index.ts, engines: bun >=1.0.0
├── tsconfig.json                 # Strict TypeScript config
├── biome.json                    # Biome linter/formatter config
├── CLAUDE.md                     # AI agent instructions (session protocol, conventions)
├── SPEC.md                       # Detailed technical specification (data model, architecture)
├── CHANGELOG.md                  # Keep a Changelog format, v0.1.0 → v0.2.5
├── V1_DONE.md                    # Post-v1 milestone notes
│
├── src/
│   ├── index.ts                  # CLI entry point, Commander setup, VERSION constant, global flags
│   ├── types.ts                  # All shared TypeScript interfaces and constants
│   ├── store.ts                  # JSONL read/write, advisory locking, atomic writes, dedup-on-read
│   ├── id.ts                     # ID generation ({project}-{4hex}, collision-checked)
│   ├── config.ts                 # YAML config load/save, findSeedsDir(), worktree resolution
│   ├── output.ts                 # Forest-palette chalk helpers, printSuccess/Error/Warning/Issue*
│   ├── yaml.ts                   # Minimal flat key-value YAML parser (~27 LOC)
│   ├── markers.ts                # Marker-delimited section utilities (for onboard command)
│   │
│   ├── commands/                 # One file per CLI command
│   │   ├── init.ts               # sd init — creates .seeds/, config.yaml, .gitignore, .gitattributes
│   │   ├── create.ts             # sd create — new issue with type/priority/description/assignee/labels
│   │   ├── show.ts               # sd show <id> — detailed issue view
│   │   ├── list.ts               # sd list — filtered listing (--status, --type, --label, --all)
│   │   ├── ready.ts              # sd ready — open issues with no unresolved blockers
│   │   ├── update.ts             # sd update <id> — field updates
│   │   ├── close.ts              # sd close <id...> — close with optional reason
│   │   ├── dep.ts                # sd dep add/remove/list — dependency management
│   │   ├── block.ts              # sd block <id> --by <blocker> — bidirectional block
│   │   ├── unblock.ts            # sd unblock <id> --from/--all — remove blockers
│   │   ├── blocked.ts            # sd blocked — list all blocked issues
│   │   ├── label.ts              # sd label add/remove/list/list-all — label management
│   │   ├── stats.ts              # sd stats — project statistics
│   │   ├── sync.ts               # sd sync — git add + commit .seeds/ (worktree-aware)
│   │   ├── tpl.ts                # sd tpl create/step/list/show/pour/status — template molecules
│   │   ├── migrate.ts            # sd migrate-from-beads — import from .beads/issues.jsonl
│   │   ├── doctor.ts             # sd doctor — health checks + --fix for auto-fixable issues
│   │   ├── prime.ts              # sd prime — AI agent context injection (full or --compact)
│   │   ├── onboard.ts            # sd onboard — insert seeds section into CLAUDE.md/AGENTS.md
│   │   ├── upgrade.ts            # sd upgrade — check/install latest from npm
│   │   └── completions.ts        # sd completions <bash|zsh|fish> — shell completion scripts
│   │
│   ├── store.test.ts             # Core data layer tests (real I/O, temp dirs)
│   ├── id.test.ts                # ID generation and collision detection tests
│   ├── yaml.test.ts              # YAML parser tests
│   ├── markers.test.ts           # Marker section utility tests
│   ├── suggestions.test.ts       # Typo suggestion (Levenshtein) tests
│   ├── timing.test.ts            # --timing flag tests
│   └── commands/
│       ├── init.test.ts
│       ├── create.test.ts
│       ├── dep.test.ts
│       ├── tpl.test.ts
│       ├── doctor.test.ts
│       ├── prime.test.ts
│       ├── onboard.test.ts
│       ├── completions.test.ts
│       ├── label.test.ts
│       ├── unblock.test.ts
│       └── sync.test.ts
│
├── scripts/
│   └── version-bump.ts           # Atomic semver bump for package.json + src/index.ts
│
├── .github/
│   └── workflows/
│       ├── ci.yml                # lint + typecheck + test on push/PR
│       └── publish.yml           # Full CI + version-check + npm publish + git tag + GitHub release
│
├── .seeds/                       # Seeds' own issue tracker (dogfooding!)
│   ├── config.yaml               # project: seeds, version: 1
│   ├── issues.jsonl              # 25+ issues tracking seeds' own development
│   ├── templates.jsonl           # Template definitions
│   └── .gitignore                # *.lock
│
├── .mulch/                       # Mulch structured expertise layer
│   ├── mulch.config.yaml         # Domains: docs, orchestration, agents, commands, architecture
│   └── expertise/                # JSONL files per domain (conventions, patterns, decisions)
│       ├── docs.jsonl
│       ├── orchestration.jsonl
│       ├── agents.jsonl
│       ├── commands.jsonl
│       ├── architecture.jsonl
│       └── cicd.jsonl
│
├── .overstory/                   # Overstory multi-agent orchestration config
│   ├── config.yaml               # maxConcurrent: 25, quality gates, worktree config
│   ├── agent-manifest.json       # 8 agent types: scout/builder/reviewer/lead/merger/coordinator/supervisor/monitor
│   ├── agent-defs/               # Per-agent role definition markdown files
│   ├── hooks.json                # Claude Code hooks: SessionStart, UserPromptSubmit, PreToolUse, etc.
│   └── groups.json               # Agent group definitions
│
├── .canopy/                      # Canopy prompt management
│   ├── config.yaml               # emitDir: .claude/commands
│   ├── prompts.jsonl             # Managed prompts
│   └── schemas.jsonl             # Schema definitions
│
└── .claude/
    ├── settings.json             # Claude Code settings
    └── commands/                 # Slash commands for Claude Code sessions
        ├── release.md            # /release — prepare a version release
        ├── pr-reviews.md         # /pr-reviews — multi-agent PR review workflow
        ├── issue-reviews.md      # /issue-reviews
        └── prioritize.md         # /prioritize
```

### Architecture

Seeds follows a simple layered architecture:

```
CLI Entry (index.ts + Commander)
         │
         ▼
Command Files (src/commands/*.ts)
         │
         ▼
Core Modules (store.ts, config.ts, id.ts, output.ts)
         │
         ▼
Storage (.seeds/issues.jsonl, .seeds/templates.jsonl)
```

**Key architectural decisions:**

1. **Each command exports a `register(program: Command)` function** — clean separation, Commander-based routing
2. **`store.ts` is the only module that touches disk** — all locking and atomic writes centralized
3. **Dedup-on-read** — last occurrence of a given ID in JSONL wins; handles `merge=union` duplicates transparently
4. **Worktree-aware config** — `findSeedsDir()` in `config.ts` resolves git worktree paths back to main repo's `.seeds/`

---

## 4. How to Run / Install It

### Install (Global CLI)

```bash
# Install globally via Bun (recommended)
bun install -g @os-eco/seeds-cli

# Or try without installing
npx @os-eco/seeds-cli --help
```

### Development Setup

```bash
git clone https://github.com/jayminwest/seeds
cd seeds
bun install           # Install dev dependencies (biome, @types/bun, typescript)
bun link              # Makes 'sd' available globally from source
```

### Environment Variables

- `NO_COLOR` — Respected for disabling ANSI color output (standard convention)
- No secrets, API keys, or external services required

### Available Commands (npm scripts)

```bash
bun test                    # Run all tests (real I/O, temp dirs, no mocks)
bun run lint                # Biome check (linting + formatting)
bun run lint:fix            # Auto-fix lint issues
bun run typecheck           # tsc --noEmit (type checking only)
bun run version:bump        # Bump version: bun run version:bump <major|minor|patch>
```

### Quality Gate (run before every commit)

```bash
bun test && bun run lint && bun run typecheck
```

### Initializing Seeds in a Project

```bash
# In your project directory
sd init                                    # Creates .seeds/, config.yaml, .gitignore, .gitattributes

# Create your first issue
sd create --title "Add retry logic" --type task --priority 1

# Find work
sd ready                                   # Open, unblocked issues
sd list                                    # All open/in-progress issues

# Work on it
sd update <id> --status in_progress
sd close <id> --reason "Done"

# Commit the tracker state
sd sync
```

### Key Global Flags (available on all commands)

| Flag | Purpose |
|------|---------|
| `--json` | Structured JSON output (for agent/script consumption) |
| `-v` / `--version` | Show version info |
| `-q` / `--quiet` | Suppress non-error output |
| `--verbose` | Extra diagnostic output |
| `--timing` | Show execution time on stderr |

---

## 5. Interesting Patterns, Features, and Capabilities

### AI-First Design

Seeds is built from the ground up to be consumed by AI agents:

- **`--json` on every command** — consistent `{ success, command, ... }` envelope. Success: `{ success: true, command: "create", id: "seeds-a1b2" }`. Error: `{ success: false, command: "create", error: "Title is required" }`.
- **`sd prime`** — outputs rich workflow context for AI sessions (full or `--compact`). Supports a custom `PRIME.md` per-project override. This is what agents run at session start to restore task-tracking context.
- **`sd onboard`** — automatically injects a seeds section into `CLAUDE.md` or `AGENTS.md` with idempotent marker-delimited sections (via `src/markers.ts`). Detects existing sections and upgrades if version number changes.
- **Bidirectional dependency tracking** — `sd dep add`, `sd block`, `sd unblock` maintain both `blocks[]` and `blockedBy[]` fields atomically; `sd ready` computes unblocked work based on this graph.

### Concurrency Safety for Multi-Agent Use

Seeds is specifically designed for parallel AI agents operating in git worktrees:

- **Advisory file locks** — `O_CREAT | O_EXCL` (atomic at OS level), 30s stale threshold, 100ms retry with random jitter, 30s timeout. Lock file: `issues.jsonl.lock`
- **Atomic writes** — All mutations write to `issues.jsonl.tmp.{random}` then `renameSync()` over the real file under lock
- **Dedup-on-read** — After `merge=union` git merge creates duplicate lines (both branches modified same issue), Seeds deduplicates by ID on every read; last occurrence wins
- **Worktree resolution** — `findSeedsDir()` + `isInsideWorktree()` in `config.ts` automatically resolves to the main repo's `.seeds/` when running inside a git worktree, preventing data isolation

### Template / Molecule System

Templates ("molecules") model multi-step workflows as a directed chain:

```bash
sd tpl create --name "scout-build-review"
sd tpl step add <tpl-id> --title "Scout: {prefix}" --priority 2
sd tpl step add <tpl-id> --title "Build: {prefix}" --priority 1
sd tpl step add <tpl-id> --title "Review: {prefix}" --priority 3
sd tpl pour <tpl-id> --prefix "Feature X"
# Creates 3 issues, step N+1 blocked by step N, tracked by shared "convoy" tag
sd tpl status <tpl-id>   # Shows completion percentage across all poured instances
```

The `{prefix}` interpolation and convoy tracking make this suitable for defining and repeating standard agent workflows.

### os-eco Ecosystem Integration

Seeds is one component of a larger AI agent toolchain. The repository itself is fully dogfooded (all issues tracked in `.seeds/issues.jsonl`):

| Tool | Role | Integration with Seeds |
|------|------|----------------------|
| **Overstory** | Multi-agent orchestration (spawns Claude Code workers in tmux+worktrees) | Wraps `sd` via `Bun.spawn(["sd", ...])` with `--json` parsing; 8 agent types defined in `.overstory/agent-manifest.json` |
| **Mulch** | Structured expertise management (JSONL knowledge base per domain) | Records conventions/patterns per domain; `mulch prime` + `mulch record` in session hooks |
| **Canopy** | Prompt management | Manages Claude slash commands, emits to `.claude/commands/` |
| **Beads** | Previous issue tracker (replaced by Seeds) | `sd migrate-from-beads` imports `.beads/issues.jsonl` |

Overstory's `hooks.json` wires deep into Claude Code's lifecycle:
- `SessionStart` → `overstory prime --agent orchestrator`
- `UserPromptSubmit` → `overstory mail check --inject` (inter-agent messaging)
- `PreToolUse[Bash]` → Blocks `git push` (merge locally first)
- `PreToolUse[*]` → Logs tool starts to overstory
- `Stop` → `mulch learn` (AI-driven expertise extraction)
- `PreCompact` → `overstory prime --compact` (context recovery before compaction)

### Claude Slash Commands

`.claude/commands/` contains rich slash command definitions for Claude Code sessions:

- **`/release`** — Automated release preparation: analyzes git log since last tag, determines semver bump, updates `CHANGELOG.md`, bumps version in both `package.json` and `src/index.ts`, updates `README.md`/`CLAUDE.md` if needed. Produces summary but does NOT commit.
- **`/pr-reviews`** — Spawns parallel sub-agents (one per PR) using Claude's Task tool; each agent runs `gh pr diff`, reviews code quality / project alignment / risk, returns structured verdict. Consolidated report with merge-order recommendation.
- **`/issue-reviews`** and **`/prioritize`** — Similar multi-agent workflows for issue triage

### Developer Experience Details

- **Typo suggestions** — Levenshtein distance on unknown commands: "Did you mean `list`?" (`src/suggestions.test.ts` validates this)
- **Shell completions** — `sd completions bash|zsh|fish` outputs completion scripts
- **`sd doctor --fix`** — Health checks for config validity, JSONL integrity, field validation, dependency graph consistency, stale locks, gitattributes presence. `--fix` auto-repairs some issues.
- **`sd upgrade`** — Self-update: checks npm registry for `@os-eco/seeds-cli`, compares with `VERSION` constant, installs via `bun install -g`. `--check` for dry-run.
- **Forest color palette** in `src/output.ts`: `brand = chalk.rgb(124, 179, 66)` (green), `accent = chalk.rgb(255, 183, 77)` (amber), `muted = chalk.rgb(120, 120, 110)` (gray)
- **Status icons**: `✓` success, `✗` error, `!` warning/blocked, `-` open issue, `>` in-progress, `x` closed

### Data Model Highlights

```typescript
interface Issue {
  id: string;           // "{project}-{4hex}" e.g. "seeds-a1b2"
  title: string;
  status: "open" | "in_progress" | "closed";
  type: "task" | "bug" | "feature" | "epic";
  priority: number;     // 0=Critical, 1=High, 2=Medium, 3=Low, 4=Backlog
  assignee?: string;    // free-text (no user management)
  description?: string;
  closeReason?: string;
  blocks?: string[];    // bidirectional dependency graph
  blockedBy?: string[];
  labels?: string[];    // added in v0.2.5
  convoy?: string;      // template instance tracking
  createdAt: string;    // ISO 8601
  updatedAt: string;
  closedAt?: string;
}
```

ID format `{project}-{4hex}` is collision-checked against existing IDs, falls back to 8 hex after 100 collisions. Matches beads' format for migration compatibility.

### Publish Pipeline

`publish.yml` is noteworthy in its completeness:
1. Run full CI gates (lint + typecheck + test)
2. Compare local version vs npm registry version (idempotent — skip if already published)
3. Verify `package.json` and `src/index.ts` versions match (prevents partial bumps)
4. `npm publish --access public` via `NPM_TOKEN` secret
5. `git tag v{version}` and push
6. Extract corresponding `CHANGELOG.md` section; create GitHub release with real notes (fallback to auto-generated)

### Self-Hosting / Dogfooding

The repository tracks its own development using Seeds. The `.seeds/issues.jsonl` file contains 25+ closed issues documenting the entire development arc: initial implementation, Commander migration, chalk migration, visual branding, CLI standards phases, and feature additions through v0.2.5.
