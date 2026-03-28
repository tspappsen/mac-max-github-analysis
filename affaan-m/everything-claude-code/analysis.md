# Repository: affaan-m/everything-claude-code

**URL:** https://github.com/affaan-m/everything-claude-code

## 1. What It Does

- A complete **agent harness performance system** — not just a config pack, but a full operational layer for AI coding assistants
- Provides battle-tested agents, skills, hooks, commands, rules, and MCP configs evolved over 10+ months of intensive daily use building real products
- Ships as a Claude Code plugin (npm: `ecc-universal`) installable via the Claude Code plugin marketplace
- Works across **Claude Code**, **Cursor**, **OpenCode**, **Codex** (app + CLI), and **Kiro** — same repo, multiple harness targets
- Won the Anthropic x Forum Ventures hackathon with zenith.chat
- v1.9.0, 50K+ GitHub stars, 6K+ forks, 30 contributors, 7 languages supported

## 2. AI/Agent Architecture

### Agents (25+ subagents)
- Each agent is a Markdown file with YAML frontmatter declaring `name`, `description`, `tools`, and `model`
- Model routing is intentional: `planner` uses `opus`, `code-reviewer` uses `sonnet`, background tasks use `haiku`
- Agent roles: planner, code-reviewer, tdd-guide, architect, chief-of-staff, loop-operator, security-reviewer, build-error-resolver, docs-lookup, harness-optimizer, and language-specific reviewers (TypeScript, Python, Go, Rust, Flutter, Java, Kotlin, C++, PyTorch)
- Agents are delegated tasks by an orchestrator (the main Claude instance) with limited tool scopes

### Hook System (26+ hook scripts)
- Claude Code's lifecycle hooks are the core automation layer: `PreToolUse`, `PostToolUse`, `SessionStart`, `Stop`, `PreCompact`, `SessionEnd`, `PostToolUseFailure`
- **PreToolUse hooks**: block `--no-verify` git flag, auto-start tmux dev servers, warn before git push, block destructive operations, enforce config protection (prevent weakening linter configs), governance capture, InsAIts AI security monitoring, MCP health checks, suggest compaction at logical intervals
- **PostToolUse hooks**: auto-format (Biome/Prettier), TypeScript type-checking after `.ts`/`.tsx` edits, console.log warnings, quality gate checks, PR URL logging, continuous learning observation capture
- **Stop hooks**: session state persistence, pattern extraction, token/cost tracking, desktop notifications (macOS)
- **SessionStart hook**: loads previous session summary, detects package manager, reports available sessions and learned skills
- Hook profile gating: `ECC_HOOK_PROFILE=minimal|standard|strict` and `ECC_DISABLED_HOOKS=...` for runtime control without editing files
- Observer loop prevention: 5-layer re-entrancy guard to prevent infinite hook→agent→hook cycles, with memory throttling and tail sampling

### Skills (80+ workflow definitions)
- Each skill is a `SKILL.md` file with structured sections: When to Use, How It Works, Examples
- Skills are markdown-based prompt engineering embedded in the harness context
- Key AI-specific skills: `eval-harness`, `verification-loop`, `skill-comply`, `continuous-learning-v2`, `search-first`, `iterative-retrieval`, `prompt-optimizer`, `regex-vs-llm-structured-text`, `cost-aware-llm-pipeline`, `safety-guard`, `deep-research`, `exa-search`
- Skills are curated in `skills/`; user-generated/imported ones go in `~/.claude/skills/`

### Continuous Learning v2.1 (Instinct Architecture)
- Observes Claude Code sessions via PreToolUse/PostToolUse hooks (100% reliable vs Stop hook only in v1)
- Extracts atomic **instincts** — small learned behaviors with confidence scoring (0.3–0.9)
- Instincts are YAML with fields: `id`, `trigger`, `confidence`, `domain`, `source`, `scope`, `project_id`
- Project-scoped instincts prevent cross-project contamination (isolated by git remote URL / repo path hash)
- Evolution pipeline: raw observations → instincts → cluster → skills/commands/agents
- Commands: `/instinct-status`, `/instinct-export`, `/instinct-import`, `/instinct-promote`, `/evolve`
- Background Haiku agent handles analysis to avoid polluting main context
- `instinct-cli.py` for import/export with `--dry-run`, `--scope`, `--min-confidence` flags

### skill-comply (Compliance Measurement)
- Automatically measures whether agents actually follow skills/rules by:
  1. Auto-generating expected behavioral sequences (specs) from any `.md` file via LLM
  2. Auto-generating test scenarios at 3 prompt strictness levels (supportive → neutral → competing)
  3. Running `claude -p` and capturing tool call traces via `stream-json`
  4. Classifying tool calls against spec steps using LLM (semantic, not regex)
  5. Checking temporal ordering deterministically
- Produces self-contained reports with compliance rates, tool call timelines, and hook promotion recommendations

### eval-harness (Eval-Driven Development)
- Implements EDD principles: define expected behavior before implementation
- Grader types: code-based (deterministic bash), LLM-based (semantic), hybrid
- Supports capability evals, regression evals, pass@k metrics
- Structured `[CAPABILITY EVAL]` / `[REGRESSION EVAL]` markdown blocks

### MCP Integrations
- `mcp-configs/mcp-servers.json` provides ready-to-use configurations for: GitHub, Firecrawl, Supabase, memory server, sequential-thinking, Vercel, Railway, Cloudflare (docs/workers/observability), ClickHouse, Exa web search, Context7 (live docs)
- Both `command` (stdio) and `http` (SSE/streaming) transport types used
- `mcp-health-check.js` hook validates MCP server health before any MCP tool call

### NanoClaw v2 (Agent REPL)
- Zero-dependency Node.js REPL wrapping `claude -p`
- Session-aware: stores conversation history as Markdown in `~/.claude/claw/`
- Features: model routing, skill hot-load, session branch/search/export/compact/metrics

### Session & State Management
- Sessions stored as `.tmp` Markdown files in `~/.claude/session-data/` (legacy: `~/.claude/sessions/`)
- Filename format: `YYYY-MM-DD-<short-id>-session.tmp`
- `session-start.js` loads last 7 days of session context into new sessions
- `session-end.js` async hook persists session state after every response
- SQLite-backed install state store with AJV schema validation
- `sessions-cli.js` for querying session history

### Security / Guardrails
- `safety-guard` skill: 3 modes (Careful, Freeze, Guard) for autonomous agent safety
- `config-protection.js` hook: blocks modifications to linter/formatter configs — steers agent to fix code instead of weakening rules
- `governance-capture.js` hook: captures secrets, policy violations, approval requests (enabled via `ECC_GOVERNANCE_CAPTURE=1`)
- `insaits-security-wrapper.js` / `insaits-security-monitor.py`: AI security monitoring on Bash/Edit/Write (enabled via `ECC_ENABLE_INSAITS=1`)
- `block-no-verify` npm package: prevents `--no-verify` git bypass
- Enterprise controls in `.claude/enterprise/controls.md`
- `security-reviewer` agent for critical security findings
- AgentShield integration: `/security-scan` runs 1282 tests, 102 rules

## 3. Key Design Decisions

- **Hook-first automation**: quality gates, learning, and safety are hook-enforced, not just prompt-suggested — the harness enforces behavior even when prompts don't
- **Confidence-scored instincts**: atomic learned behaviors decay/promote based on frequency and project scope — prevents noisy pattern injection
- **Model routing by task type**: Opus for planning/architecture, Sonnet for code review, Haiku for background/analysis — explicit cost vs quality tradeoff
- **Eval-first mindset**: `skill-comply` measures actual compliance, not just intent; EDD treats evals as "unit tests of AI development"
- **Cross-harness parity**: same skills/rules shipped to Claude Code, Cursor, OpenCode, Codex, Kiro via separate adapter directories (`.cursor/`, `.codex/`, `.kiro/`, `.opencode/`)
- **Selective install**: manifest-driven pipeline (`install-plan.js` + `install-apply.js`) with state tracking for incremental updates
- **Observer loop prevention**: hooks that trigger agent calls that trigger hooks is a real failure mode — 5-layer guard with re-entrancy flags
- **Project-scoped instincts**: isolates learned patterns per repository to prevent React patterns leaking into Python projects
- **Config protection**: blocking agents from weakening linter configs is a novel guardrail pattern — agents tend to silence errors rather than fix code

## 4. Project Structure

```
everything-claude-code/
├── agents/              # 25+ subagent definitions (YAML frontmatter + markdown)
├── skills/              # 80+ workflow skills (SKILL.md format)
├── commands/            # 60+ slash commands (markdown with description frontmatter)
├── hooks/
│   ├── hooks.json       # Claude Code hook config (PreToolUse/PostToolUse/Session/Stop)
│   └── README.md
├── rules/               # Per-language always-on guidelines
│   ├── common/          # Security, testing, patterns, git-workflow, performance
│   ├── typescript/      # TS-specific coding-style, hooks, patterns, security, testing
│   ├── python/          # Python-specific rules (same structure)
│   ├── golang/          # Go-specific rules
│   ├── rust/            # Rust-specific rules
│   ├── java/            # Java-specific rules
│   ├── php/             # PHP/Laravel rules
│   ├── perl/            # Perl rules
│   └── kotlin/          # Kotlin/Android/KMP rules
├── mcp-configs/         # MCP server configurations (GitHub, Supabase, Exa, etc.)
├── scripts/
│   ├── hooks/           # 26 Node.js hook scripts
│   ├── lib/             # session-manager, package-manager, utils, install-state, skill-evolution
│   ├── claw.js          # NanoClaw v2 agent REPL
│   ├── harness-audit.js # Deterministic harness scoring
│   ├── install-plan.js  # Manifest-driven selective install planner
│   ├── install-apply.js # Install executor
│   └── orchestrate-worktrees.js
├── contexts/            # dev.md, research.md, review.md context files
├── .claude-plugin/      # Claude Code plugin manifest (plugin.json, marketplace.json)
├── .claude/             # Claude Code specific: rules, skills, commands, identity, MCP
├── .opencode/           # OpenCode plugin (TypeScript, 7 custom tools, hooks)
├── .cursor/             # Cursor rules, hooks, skills
├── .codex/              # Codex TOML agents and config
├── .kiro/               # Kiro agents (JSON+MD), hooks, skills
├── tests/               # 997+ Node.js tests (hooks, lib, scripts, integration, CI)
└── install.sh / install.ps1
```

## 5. Tech Stack

- **Node.js** — primary scripting runtime for hooks, session management, install pipeline, CLI tools
- **Python** — `skill-comply` eval harness, `instinct-cli.py`, continuous learning scripts, InsAIts security monitor
- **TypeScript** — OpenCode plugin, 7 custom tools (`run-tests`, `check-coverage`, `security-audit`, `format-code`, `lint-check`, `git-summary`, `check-coverage`)
- **Shell** — bash install scripts, some hook wrappers, skill utility scripts
- **YAML** — instinct definitions, agent configs (`.agents/` openai.yaml for Codex)
- **JSON** — hooks.json, plugin.json, MCP configs, install state schema (AJV validated)
- **TOML** — Codex agent configs (`.codex/agents/`)
- **SQLite** — install state store, session state
- **MCP protocol** — external tool integrations via Model Context Protocol
- **Languages targeted**: TypeScript, Python, Go, Rust, Java, PHP, Perl, Kotlin, C++, Swift

## 6. Patterns Worth Borrowing

- **Instinct-based continuous learning**: atomic confidence-scored behaviors that auto-extract from sessions, cluster over time, and promote into reusable skills — far more granular than monolithic skill extraction
- **Hook-enforced quality gates**: running type-check and formatter as PostToolUse hooks means code quality is enforced by the harness, not just suggested in prompts
- **Config protection hook**: blocking agents from weakening ESLint/Biome configs is a subtle but high-value guardrail — agents consistently try to silence errors rather than fix code
- **skill-comply for compliance measurement**: LLM-classified tool call traces measured against auto-generated behavioral specs across 3 prompt strictness levels — a genuine eval framework for agent behavior, not just output quality
- **Model routing in agent definitions**: explicit `model: opus|sonnet|haiku` per agent makes cost/quality tradeoffs transparent and auditable
- **Observer loop prevention**: detailed re-entrancy guard with 5 layers is a solved problem worth copying directly for any hook-based system
- **Project-scoped vs global memory**: applying instincts/patterns only to the project where they were learned prevents cross-contamination — essential for multi-project users
- **Eval-Driven Development (EDD)**: writing behavioral specs before implementation and measuring with pass@k metrics applied to agent sessions, not just unit tests
- **Session persistence via Stop/SessionStart hooks**: injecting the last session's summary as context on new sessions is a practical cross-session memory solution without external infrastructure
- **Harness audit scoring**: `harness-audit.js` produces a deterministic score across 7 categories (tool coverage, context efficiency, quality gates, memory persistence, eval coverage, security guardrails, cost efficiency) — a useful self-assessment framework

## 7. Gaps, Risks, and Limitations

- **Claude Code internals dependency**: deeply coupled to Claude Code's hook event names, stdin/stdout JSON protocol, `transcript_path`, and plugin root resolution — any breaking change in the harness would require significant rework
- **Config complexity**: 7+ harness targets × 12 languages × 5 rule types = combinatorial maintenance burden; some harness adapters (Kiro, Codex) appear less mature than Claude Code
- **No LLM client code**: this is a configuration/orchestration layer, not an LLM SDK — it orchestrates Claude Code's built-in capabilities rather than calling APIs directly (which is the right design but worth noting for those expecting an SDK)
- **Continuous learning requires rich sessions**: instinct extraction depends on tool call frequency and session length; sparse sessions won't generate useful patterns
- **Memory explosion risk**: acknowledged in release notes — the observer hooks are async with throttling, but high-frequency tool use in long sessions can still cause issues; tail sampling is the mitigation
- **Install path assumptions**: many scripts hardcode `~/.claude/` paths; team/enterprise setups with non-standard Claude directories may need patching
- **Python dependency management**: `skill-comply` uses `uv run python -m scripts.run` — requires `uv` and Python in path, which may not be universal
- **No RAG pipeline**: there's no vector store or embedding-based retrieval — skills are statically loaded into context; as the skill library grows (80+ and counting), context pressure becomes a real concern
- **Cross-harness parity is aspirational**: the OpenCode and Codex adapters have materially fewer features than the Claude Code adapter; "works across all harnesses" is directionally true but not feature-parity
- **Governance capture is opt-in**: `ECC_GOVERNANCE_CAPTURE=1` and `ECC_ENABLE_INSAITS=1` are disabled by default — the most powerful security features require explicit enablement

## 8. How to Run It

### Quickstart (Claude Code plugin)
```bash
# Install via Claude Code plugin system
/plugin marketplace add affaan-m/everything-claude-code
/plugin install everything-claude-code@everything-claude-code
```

### Full Installation (recommended for rules/hooks)
```bash
git clone https://github.com/affaan-m/everything-claude-code.git
cd everything-claude-code
npm install

# macOS/Linux — full profile
./install.sh --profile full

# Or language-specific
./install.sh typescript python golang

# Windows
.\install.ps1 --profile full
```

### Targeting specific harnesses
```bash
./install.sh --target cursor typescript
./install.sh --target antigravity typescript
```

### NanoClaw REPL
```bash
node scripts/claw.js
```

### Running tests
```bash
node tests/run-all.js
# or
npm test
```

### Harness audit
```bash
node scripts/harness-audit.js
node scripts/harness-audit.js --format json --scope hooks
```

### Skill compliance measurement
```bash
uv run python -m scripts.run ~/.claude/rules/common/testing.md
uv run python -m scripts.run --dry-run skills/search-first/SKILL.md
```

### Environment variables
- `ECC_HOOK_PROFILE=minimal|standard|strict` — controls which hooks activate
- `ECC_DISABLED_HOOKS=pre:bash:tmux-reminder,stop:cost-tracker` — disable specific hooks
- `ECC_GOVERNANCE_CAPTURE=1` — enable governance event capture
- `ECC_ENABLE_INSAITS=1` — enable AI security monitoring
- `CLAUDE_PACKAGE_MANAGER=npm|pnpm|yarn|bun` — override package manager detection
- `CLAW_MODEL=opus|sonnet|haiku` — default model for NanoClaw REPL
