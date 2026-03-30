# Repository: oh-my-mermaid/oh-my-mermaid

**URL:** https://github.com/oh-my-mermaid/oh-my-mermaid

## 1. What It Does

`omm` (oh-my-mermaid) is an **architecture documentation tool for AI-assisted ("vibe-coded") codebases**. It solves the problem that AI writes code faster than humans can understand it — leaving developers with a black-box codebase they don't comprehend.

The tool bridges that gap by having an AI agent (`/omm-scan` skill) analyze the codebase and generate **perspectives** — multiple architectural lenses (overall architecture, data flow, request lifecycle, etc.) stored as Mermaid diagrams + rich markdown fields in a `.omm/` directory. A local HTTP viewer (localhost:3000) renders the nested diagrams interactively. Docs can optionally be pushed to a cloud service (ohmymermaid.com) for team sharing.

**Core problem solved:** AI vibe-coding produces working code but destroys architectural legibility. `omm` restores legibility by generating human-readable architecture docs as a first-class output of the AI coding process.

## 2. AI/Agent Architecture

### Skill-Based Agent Pattern

`omm` does not embed an LLM directly. Instead, it ships **Skill files** — markdown documents that get registered into AI coding tools (Claude Code, Cursor, Codex, OpenClaw, Antigravity). When a user types `/omm-scan`, the AI tool reads the skill file as a system prompt and executes the prescribed workflow.

### omm-scan Skill Workflow

The `/omm-scan` skill is a **structured agentic prompt** that instructs the LLM to:

1. **Explore** — use Glob + Read to understand the codebase (package manifests, entry points, routes, services)
2. **Select perspectives** — choose from a curated catalog of 12 perspective types (overall-architecture, data-flow, request-lifecycle, etc.) based on what exists
3. **Generate with recursive drill-down:**
   - Write the top-level perspective diagram (Mermaid `graph LR`)
   - Write all 6 supporting fields: description, context, constraint, concern, todo, note
   - For every node in the diagram: analyze the code it represents, decide leaf vs group
   - If group: write a sub-diagram and recurse deeper
   - This creates a filesystem tree under `.omm/` mirroring the architecture

4. **Materialize via CLI** — the AI calls `omm write <path> <field>` shell commands to persist results. The CLI is the deterministic storage layer; the AI is the analysis engine.

### Agent/LLM Boundary

Clear separation:
- **LLM side:** Analysis, diagram generation, field content writing, recursion decisions
- **CLI side:** File I/O, meta tracking, diff computation, viewer serving, cloud push

The skill acts as a **structured prompt + protocol** telling the AI exactly which CLI commands to call and in what order. No LLM API is embedded in the CLI itself.

### Prompt Engineering

The skill file is the prompt. Key patterns:
- **Perspective catalog** with explicit `When to create` / `What it answers` columns — guides the model to select only meaningful perspectives, not force all
- **Recursive drill-down algorithm** described step-by-step with code examples
- **Language configuration** (`omm config language`) — content written in configured language (ko, ja, zh, en), with IDs always English kebab-case
- **7 structured fields per element** — each field targets a different concern (description = what, context = why, constraint = rules, concern = risks, todo = future work, note = misc)

## 3. Key Design Decisions

1. **Filesystem IS the data model** — `.omm/<perspective>/<child>/<grandchild>/` — no database. Nesting is implicit in directory structure. The viewer reads the filesystem to determine groups vs leaves.

2. **CLI wraps AI, not vice-versa** — The AI calls shell commands (`omm write ...`), not an API. This makes the AI's output observable, diffable, and git-trackable.

3. **7 fields per element** — Deliberate separation of concerns: `description` (what), `context` (why/decisions), `constraint` (rules), `concern` (risks/debt), `todo` (work items), `note` (misc). Pushes the AI to produce richer structured analysis rather than a blob.

4. **Diagram diff tracking** — `meta.yaml` per element stores `prev_diagram`, enabling `omm diff <class>` to show what changed between scans. Git-native history also available.

5. **Platform abstraction** — `Platform` interface (detect/isSetup/setup/teardown) allows the same skill to be installed across Claude Code, Cursor, Codex, OpenClaw, Antigravity via a single `omm setup` command.

6. **No bundled LLM** — Lightweight CLI (only `yaml` as runtime dependency). The AI inference cost is paid by whatever tool the user already uses.

## 4. Project Structure

```
oh-my-mermaid/
├── src/
│   ├── cli.ts                    # Entry point, command dispatch
│   ├── types.ts                  # Core types (Field, ClassMeta, ClassData, etc.)
│   ├── commands/                 # One file per CLI command
│   │   ├── setup.ts              # Platform registration
│   │   ├── view.ts               # Start local viewer
│   │   ├── push.ts, pull.ts      # Cloud sync
│   │   ├── diff.ts, refs.ts      # Analysis commands
│   │   └── read-write.ts         # omm write/read core
│   ├── lib/
│   │   ├── store.ts              # All .omm/ filesystem operations
│   │   ├── cloud.ts              # API client (credentials, HTTP)
│   │   ├── diff.ts               # Mermaid diagram diff logic
│   │   ├── refs.ts               # Cross-element reference resolution
│   │   ├── validate.ts           # Diagram validation
│   │   ├── meta.ts               # meta.yaml management
│   │   └── platforms/            # claude, cursor, codex, openclaw, antigravity
│   └── server/
│       ├── index.ts              # HTTP server (localhost:3000)
│       ├── api.ts                # REST endpoints for viewer
│       ├── watcher.ts            # SSE filesystem watcher (live reload)
│       └── viewer.html           # Single-file viewer app
├── skills/
│   ├── omm-scan/SKILL.md         # Main AI agent skill (the "system prompt")
│   ├── omm-push/SKILL.md         # Cloud push skill
│   └── omm-view/SKILL.md         # View skill
├── .claude-plugin/               # Claude Code plugin manifest
│   ├── plugin.json
│   └── marketplace.json
├── .omm/                         # omm scanned itself (dog-fooded!)
│   ├── overall-architecture/
│   ├── data-flow/
│   ├── command-surface/
│   └── config.yaml
└── docs/
    └── ROADMAP.md
```

## 5. Tech Stack

| Layer | Technology |
|-------|-----------|
| **Language** | TypeScript 5.x |
| **Runtime** | Node.js ≥18 (ESM modules) |
| **Build** | tsup (esbuild-based) |
| **Test** | Vitest |
| **Runtime deps** | `yaml` (YAML parsing only) |
| **Dev deps** | tsup, typescript, vitest, @types/node |
| **Viewer** | Vanilla HTML/JS (single file), SSE for live reload |
| **Cloud** | REST API to ohmymermaid.com, JWT tokens in `~/.omm/credentials.json` |
| **Diagram format** | Mermaid (.mmd files) |
| **Package manager** | npm |
| **Distribution** | npm global install (`npm install -g oh-my-mermaid`) |

## 6. Patterns Worth Borrowing

- **Skill-as-prompt pattern** — Ship agentic workflows as markdown skill files that get registered into whatever AI tool the user runs. No vendor lock-in, no LLM API key management, reusable across platforms.

- **CLI as AI's output materialization layer** — AI calls shell commands to persist results. Makes AI outputs observable, debuggable, and git-diffable without building a complex API.

- **Structured field decomposition** — Instead of asking AI for a blob, define 7 named fields per artifact (what/why/constraints/risks/todos/notes). Dramatically improves output quality and navigability.

- **Filesystem as data model** — Use directory nesting as the implicit schema. Zero migration cost, git-native, universally readable.

- **Recursive agentic drill-down** — Prompt the AI to recursively analyze every node, deciding leaf vs group, until it has covered the full tree. Produces deep, exhaustive documentation in one pass.

- **Platform interface pattern** — Thin interface (detect/isSetup/setup/teardown) for installing into multiple AI coding tools without code duplication.

## 7. Gaps, Risks, and Limitations

### Gaps
- **No LLM observability** — No tracing, logging, or cost tracking of AI calls. The AI's reasoning is opaque; only its CLI output is captured.
- **No incremental scanning** — Full rescan on every `/omm-scan` call. Planned but not implemented. Large codebases will be expensive.
- **No conflict resolution** — If two scans run concurrently or a human edits `.omm/` files, there's no merge strategy.
- **Single-agent only** — Roadmap mentions sub-agent pipeline (parallel per-perspective analysis) but not yet implemented.
- **No embedding/semantic search** — Viewer is filesystem-based navigation only. No "find where auth happens" search. Planned in roadmap.
- **No validation of AI-generated Mermaid** — The `validate.ts` module exists but AI-generated diagrams aren't validated before write.

### Risks
- **AI skill file is the whole system** — If the LLM doesn't follow the skill instructions precisely, output quality degrades badly. No programmatic enforcement of the protocol.
- **`.omm/` becoming stale** — No automatic trigger when code changes. Docs will drift unless developers actively re-run `/omm-scan`.
- **Cloud dependency for sharing** — The public sharing feature requires ohmymermaid.com which is an external SaaS. Self-hosting not documented.
- **`omm write` via stdin** — The AI uses heredoc stdin to pipe content; if the LLM truncates content or makes a shell escaping error, content gets silently corrupted.
- **No auth/sandboxing on local server** — `omm view` serves `.omm/` contents over HTTP with no auth. Acceptable for localhost but a risk if port-forwarded.

### Unknowns
- Quality/consistency of AI-generated diagrams varies by model and codebase size
- How the cloud push/pull handles large `.omm/` trees with many nested elements
- Whether skill files work reliably across all 5 supported platforms or mainly tested with Claude Code

## 8. How to Run It

### Install & Setup
```bash
npm install -g oh-my-mermaid
omm setup                   # auto-detect and configure AI tools
```

### Generate Architecture Docs
Open your AI coding tool (e.g., Claude Code) in the project directory:
```
/omm-scan
```

### View Results
```bash
omm view                    # opens http://localhost:3000
```

### Other CLI Commands
```bash
omm list                    # list all perspectives
omm show <perspective>      # show all fields for a perspective
omm diff <perspective>      # show diagram changes since last scan
omm config language ko      # set content language (ko/ja/zh/en)
omm update                  # update to latest version
omm login && omm push       # push to cloud
```

### Development
```bash
git clone https://github.com/oh-my-mermaid/oh-my-mermaid.git
cd oh-my-mermaid
npm install
npm run build               # tsup build → dist/
npm test                    # vitest run
npm run dev                 # tsup --watch
```
