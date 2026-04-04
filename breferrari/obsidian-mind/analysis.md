# Repository: breferrari/obsidian-mind

**URL:** https://github.com/breferrari/obsidian-mind

## 1. What It Does

obsidian-mind solves the **stateless AI assistant problem**: Claude Code forgets everything between sessions, forcing users to re-establish context repeatedly. The repo is a ready-to-use Obsidian vault template that acts as Claude's external brain — a structured knowledge graph of work notes, decisions, people, incidents, and performance evidence, combined with a complete Claude Code configuration (hooks, subagents, slash commands, skills) that injects context automatically at session start and routes every user message to the right vault location. The result: Claude starts every session already knowing your goals, active projects, recent changes, and team dynamics, and every conversation incrementally builds the graph rather than evaporating. There is **no application code calling LLM APIs** — the entire system runs on top of Claude Code as a configuration layer.

---

## 2. AI/Agent Architecture

### LLM Runtime
The system **is** Claude Code — Anthropic's agentic CLI. No LLM SDK calls exist in the repo. All intelligence runs through Claude's native tools (Read, Write, Edit, Grep, Glob, Bash, WebFetch, etc.) governed by `.claude/` configuration.

### Five Lifecycle Hooks (`.claude/settings.json`)
Hooks fire shell scripts at key Claude Code lifecycle events:

| Hook | Script | Effect |
|------|--------|--------|
| `SessionStart` | `session-start.sh` | Runs `qmd update`, then injects: today's date, North Star (first 30 lines), recent git log, open Obsidian tasks, active project list, full vault file listing — all as injected context |
| `UserPromptSubmit` | `classify-message.py` | Regex-classifies each message (decision / incident / 1:1 / win / architecture / person context / project update) and emits `additionalContext` routing hints back to Claude |
| `PostToolUse` | `validate-write.py` | After any Write/Edit/MultiEdit to `.md` files, checks for YAML frontmatter (`date`, `description`, `tags`), and presence of `[[wikilinks]]`; emits `additionalContext` hygiene warnings if violated |
| `PreCompact` | `pre-compact.sh` | Before context compaction, backs up the session JSONL transcript to `thinking/session-logs/` (keeps last 30) |
| `Stop` | inline `echo` | Prints end-of-session checklist to stdout |

### Subagents (`.claude/agents/`, 9 agents)
Each agent is a markdown file with YAML frontmatter (`name`, `description`, `tools`, `model`, `maxTurns`, `skills`) and a freeform instruction body. Claude Code runs them in isolated context windows — they don't pollute the main conversation context. The orchestration is explicit: commands name which subagents to launch and whether to run them in parallel.

| Agent | Key Behavior |
|-------|-------------|
| `brag-spotter` | Scans work notes, git log, incidents for uncaptured wins; uses `qmd query` then validates against brag doc |
| `context-loader` | Given a topic, runs `qmd query + vsearch`, reads backlinks via `obsidian backlinks`, synthesizes a briefing |
| `cross-linker` | Builds link-target lookup from all people/teams/competency/project notes; greps for unlinked mentions; optionally adds `[[wikilinks]]` |
| `people-profiler` | Calls `slack_read_user_profile` / `slack_search_users` MCP tools; creates/updates `org/people/` notes; updates `People & Context.md` index |
| `review-prep` | Aggregates brag doc, decisions, incidents, competency backlinks, 1:1 notes, PR evidence for a date range |
| `slack-archaeologist` | Fully reconstructs Slack channels/DMs/threads using `slack_read_channel` / `slack_read_thread` MCP tools; paginates until all messages fetched |
| `vault-librarian` | Runs `obsidian orphans`, `obsidian unresolved`, validates frontmatter, flags stale active notes, verifies index consistency |
| `review-fact-checker` | Extracts every factual claim from a review draft (numbers, timelines, attributions, comparisons); verifies each against vault sources |
| `vault-migrator` | Two-mode: (A) classify source vault files without writing; (B) execute an approved migration plan — transform frontmatter, fix wikilinks, route to correct folders |

### Slash Commands (`.claude/commands/`, 15 commands)
Markdown files read by Claude Code as command implementations. Some are simple prompts (`/standup`, `/dump`); others are orchestration scripts that name subagents to launch in parallel (`/incident-capture` → `slack-archaeologist` + `people-profiler` in parallel; `/vault-audit` → `vault-librarian` then `cross-linker`).

### Message Classification Loop
Every user message triggers `classify-message.py` (Python, regex, no ML). It emits `additionalContext` with routing hints like `"DECISION detected — consider creating a Decision Record in work/active/"`. This nudges Claude toward correct filing without requiring the user to know vault conventions.

### Skills (`.claude/skills/`)
Five skill modules loaded via Claude Code's Skill tool before relevant operations:
- `obsidian-markdown` — wikilinks, embeds, callouts, YAML properties syntax
- `obsidian-cli` — CLI command reference for the running Obsidian instance
- `obsidian-bases` — `.base` database-view file format
- `json-canvas` — `.canvas` visual map format
- `qmd` — semantic search proactive-use protocol (when to query vs. vsearch vs. keyword search)

### Memory Architecture
- **`~/.claude/projects/<encoded-path>/memory/MEMORY.md`** — auto-loaded by Claude Code at session start; contains only a routing table (topic → vault path). Never stores actual knowledge.
- **`brain/*.md`** — all durable knowledge (Memories, Key Decisions, Patterns, Gotchas, North Star, Skills). Git-tracked, Obsidian-browsable, linked.
- Rule enforced in both CLAUDE.md and memory-template.md: never create additional files in `~/.claude/projects/` — they aren't version-controlled.

### QMD Semantic Search
Optional tool (`npm install -g @tobilu/qmd`). Hybrid BM25 + vector + LLM reranking. Used proactively:
- Before reading any file (check for existing content first)
- Before creating a new note (deduplication)
- After creating a note (find notes that should link to it)
- `qmd update` runs at every `SessionStart` for incremental reindex; `qmd embed` regenerates embeddings after bulk changes.

---

## 3. Key Design Decisions

1. **Configuration-as-product, not code-as-product.** The entire system is Claude Code configuration (markdown, shell scripts, Python hooks). This makes it forkable/customizable by non-engineers and lets it evolve with Claude Code's capabilities without maintaining SDK dependencies.

2. **Vault-first memory over `~/.claude/` memory.** All durable knowledge lives in git-tracked Obsidian markdown. The `MEMORY.md` in `~/.claude/` is a pointer index only. This means memories survive machine changes, are human-readable in Obsidian, and are part of the graph (they can be wikilinked and backlinked).

3. **Graph-first organization.** A note lives in exactly ONE folder (its home) but links to MANY notes (its context). "A note without links is a bug" is a first-class rule, enforced by the `PostToolUse` hook. Folders are for browsing convenience; the graph is the real structure.

4. **Subagents for context isolation.** Heavy operations (Slack reconstruction, vault migration, PR deep scans) run in isolated context windows with defined `maxTurns` budgets. This prevents context window pollution and makes operations composable.

5. **Atomic notes + description field.** Every note gets a ~150-char `description` frontmatter field. This enables "progressive disclosure" — agents can read `qmd multi-get` results or Bases views showing descriptions without loading full note content.

6. **Pre-classification hook.** Classifying messages on `UserPromptSubmit` rather than relying on Claude to infer routing intent reduces hallucinated file placements and keeps conventions consistent across sessions.

7. **Parallel subagent invocation.** Commands like `/incident-capture` explicitly say "Launch both subagents in parallel" — a deliberate latency optimization given Claude Code's subagent isolation model.

8. **`validate-write.py` as a forcing function.** Rather than documenting conventions and hoping they're followed, the PostToolUse hook actively rejects writes that violate frontmatter and wikilink requirements. This makes conventions self-enforcing.

---

## 4. Project Structure

```
obsidian-mind/
├── CLAUDE.md                   ← 285-line operating manual; Claude reads every session
├── README.md                   ← Product docs (public-facing)
├── vault-manifest.json         ← Template metadata, version fingerprints, migration boundaries
├── CHANGELOG.md                ← Version history (v1–v3.3)
├── Home.md                     ← Vault dashboard (embeds all Bases views)
│
├── .claude/                    ← [AI CORE] All Claude Code config
│   ├── settings.json           ← Hook definitions (5 hooks → 5 scripts)
│   ├── memory-template.md      ← Template for ~/.claude/MEMORY.md
│   ├── agents/                 ← [SUBAGENTS] 9 isolated context agents
│   │   ├── brag-spotter.md
│   │   ├── context-loader.md
│   │   ├── cross-linker.md
│   │   ├── people-profiler.md
│   │   ├── review-fact-checker.md
│   │   ├── review-prep.md
│   │   ├── slack-archaeologist.md
│   │   ├── vault-librarian.md
│   │   └── vault-migrator.md
│   ├── commands/               ← [SLASH COMMANDS] 15 command implementations
│   │   ├── standup.md, dump.md, wrap-up.md, weekly.md
│   │   ├── capture-1on1.md, incident-capture.md
│   │   ├── slack-scan.md, peer-scan.md
│   │   ├── review-brief.md, self-review.md, review-peer.md
│   │   ├── vault-audit.md, vault-upgrade.md, project-archive.md
│   │   └── humanize.md
│   ├── scripts/                ← [HOOKS] Lifecycle automation
│   │   ├── session-start.sh    ← Context injection at startup
│   │   ├── classify-message.py ← Per-message routing hint injection
│   │   ├── validate-write.py   ← Post-write frontmatter/wikilink validation
│   │   ├── pre-compact.sh      ← Transcript backup before compaction
│   │   ├── charcount.sh        ← Utility: character counting for review limits
│   │   └── find-python.sh      ← Utility: cross-platform Python discovery
│   └── skills/                 ← [SKILLS] Domain knowledge modules
│       ├── obsidian-markdown/  ← Wikilinks, callouts, properties syntax
│       ├── obsidian-cli/       ← Running Obsidian CLI command reference
│       ├── obsidian-bases/     ← .base file format + functions reference
│       ├── json-canvas/        ← .canvas visual map format
│       ├── qmd/                ← Semantic search protocol
│       └── defuddle/           ← Web page → markdown extraction
│
├── brain/                      ← [AI MEMORY] Claude's operational knowledge
│   ├── North Star.md           ← Goals; read at every session start
│   ├── Memories.md             ← Index of memory topics
│   ├── Key Decisions.md        ← Significant decisions
│   ├── Patterns.md             ← Recurring patterns
│   ├── Gotchas.md              ← Known failure modes
│   └── Skills.md               ← Custom workflows/commands registry
│
├── work/                       ← [USER CONTENT] Project work
│   ├── active/                 ← Current projects (1–3 files target)
│   ├── archive/YYYY/           ← Completed work by year
│   ├── incidents/              ← Incident docs
│   ├── 1-1/                    ← 1:1 meeting notes
│   └── Index.md                ← Map of Content (MOC)
│
├── org/                        ← [USER CONTENT] Org knowledge
│   ├── people/                 ← One note per person
│   ├── teams/                  ← One note per team
│   └── People & Context.md     ← MOC
│
├── perf/                       ← [USER CONTENT] Performance tracking
│   ├── Brag Doc.md             ← Running win log with evidence links
│   ├── brag/                   ← Quarterly brag notes
│   ├── competencies/           ← One note per competency (link targets)
│   └── evidence/               ← PR deep scans, data extracts
│
├── bases/                      ← [OBSIDIAN] Dynamic database views
│   ├── Work Dashboard.base
│   ├── Incidents.base
│   ├── People Directory.base
│   ├── 1-1 History.base
│   ├── Review Evidence.base
│   ├── Competency Map.base
│   └── Templates.base
│
├── templates/                  ← Obsidian note templates with YAML frontmatter
├── thinking/                   ← Scratchpad (promote findings, then delete)
└── reference/                  ← Codebase knowledge, architecture maps
```

---

## 5. Tech Stack

- **AI Runtime**: [Claude Code](https://docs.anthropic.com/en/docs/claude-code) — the LLM is Anthropic Claude (Sonnet by default for subagents per `model: sonnet` frontmatter)
- **Semantic Search**: [QMD](https://github.com/tobi/qmd) (`@tobilu/qmd`) — hybrid BM25 + vector + LLM reranking; optional but recommended
- **Note Graph**: [Obsidian](https://obsidian.md) 1.12+ — vault, wikilink graph, Bases views, CLI (`obsidian` command)
- **Skills Framework**: [kepano/obsidian-skills](https://github.com/kepano/obsidian-skills) — official Obsidian skill modules for Claude
- **Hook Scripts**: Python 3.8+ (classify-message.py, validate-write.py, pre-compact.sh helpers), Bash
- **Slack Integration**: MCP Slack tools (`slack_read_channel`, `slack_read_thread`, `slack_read_user_profile`, `slack_search_users`) — used by `slack-archaeologist` and `people-profiler` agents
- **GitHub Integration**: `gh` CLI — used by `/peer-scan` command for PR fetching
- **Web Extraction**: [defuddle](https://github.com/kepano/defuddle) — web page to clean markdown
- **Version Control**: Git — vault is a git repo; git history used in standup and session context
- **No vector database** — QMD handles its own embedding storage locally
- **No dedicated LLM API calls** — everything routes through Claude Code's native tool use

---

## 6. Patterns Worth Borrowing

1. **Hook-driven context injection** (`session-start.sh`). Instead of prompting users to "paste your goals," auto-inject structured context at session start. The pattern: serialize relevant state (goals, active work, recent changes, file listing) into a `## Session Context` markdown block, delivered via `hookSpecificOutput.additionalContext`. Reusable for any domain-specific agent that needs warm context.

2. **Per-message classification hook** (`classify-message.py`). A lightweight regex classifier runs on every message and emits routing hints. No LLM call needed — pure pattern matching for high-confidence signals. The `additionalContext` mechanism feeds hints back into Claude's context window non-intrusively. This is the right architecture for "guide without constraining" intent routing.

3. **Post-tool-use validation as constraint enforcement** (`validate-write.py`). Validating outputs immediately after writes (not after the full session) gives Claude immediate feedback to self-correct before moving on. Any schema compliance requirement (frontmatter, required fields, link presence) fits this pattern.

4. **Subagent YAML frontmatter with `maxTurns` + `skills`**. Declarative agent configuration in markdown — tools allowed, model, turn budget, skills to load. Low-overhead way to define specialized agents with hard resource limits. The skills list means each agent gets only the domain knowledge it needs.

5. **Two-mode agent design** (`vault-migrator.md`). Mode A: analysis/classification (read-only, returns structured plan). Mode B: execution (write, given approved plan). This human-in-the-loop approval gate before destructive operations is a clean pattern for any agentic migration or bulk-edit system.

6. **`description` field for progressive disclosure**. Every note carries a ~150-char `description` in YAML frontmatter. This allows batch retrieval of many notes' summaries (via `qmd multi-get` or Bases views) without loading full content. Essential for agents that need to survey large note graphs without burning context tokens.

7. **Vault-first memory with pointer index** (memory-template.md). The `~/.claude/MEMORY.md` is a routing table, not storage. Actual memories live in version-controlled, human-readable markdown. The pattern decouples "where Claude looks first" from "where knowledge actually lives" — enables git-tracked, portable, linkable memory.

8. **Parallel subagent orchestration** (`incident-capture.md`). Explicit instruction to "launch both subagents in parallel" for independent heavy tasks. When designing multi-agent commands, identify which agents have no data dependency on each other and make parallelism a first-class concern.

---

## 7. Gaps, Risks, and Limitations

1. **Claude Code vendor lock-in.** The entire system assumes Claude Code's specific hook system, subagent YAML format, Skill tool, and `additionalContext` API. None of this is portable to LangChain, AutoGen, or custom agent frameworks. If Anthropic changes these APIs, the system breaks.

2. **No embedding model control.** QMD's vector embeddings are opaque — the repo doesn't specify which embedding model QMD uses, its dimensionality, or its cost profile. At vault scale (hundreds of notes), cold `qmd embed` runs could be slow/expensive.

3. **Regex classification is brittle.** `classify-message.py` uses simple `\b`-bounded regex to detect intent signals. It produces false positives (e.g., "the system is down" triggers INCIDENT) and false negatives for unconventional phrasing. There's no confidence scoring or fallback.

4. **MCP tool availability assumed.** `slack-archaeologist` and `people-profiler` assume `slack_read_channel`, `slack_read_thread`, `slack_read_user_profile`, `slack_search_users` MCP tools are available in the Claude Code environment. There's no graceful degradation if Slack MCP isn't configured.

5. **`maxTurns` limits may be insufficient.** Agents like `vault-migrator` (maxTurns: 50) and `slack-archaeologist` (maxTurns: 40) could exceed their budgets on large vaults or long Slack threads. There's no continuation or checkpointing mechanism.

6. **No conflict resolution for concurrent edits.** The system is single-user and single-session by design. If two Claude Code sessions run simultaneously (possible with multiple terminals), there's no locking on vault files.

7. **Session transcript backup is best-effort.** `pre-compact.sh` uses `cp` — if the transcript path is wrong or the file doesn't exist, the hook silently exits 0. Lost compaction history can't be recovered.

8. **Thinking notes as debt.** The "scratchpad → promote → delete" workflow for `thinking/` depends entirely on Claude remembering to delete. In practice, thinking notes accumulate. The vault-audit command addresses this partially but doesn't automatically delete.

9. **No chunking strategy for large notes.** The QMD skill instructs proactive search, but doesn't define note size limits or chunking. Very long notes (e.g., a full incident RCA) degrade search quality. The `description` field mitigates this but doesn't fully solve it.

10. **Competency linking requires manual annotation.** The brag/competency evidence trail only works if Claude correctly adds competency wikilinks to work notes' `## Related` sections. There's no automated competency inference from note content.

---

## 8. How to Run It

### Prerequisites
- [Obsidian](https://obsidian.md) 1.12+ (for CLI support)
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (`npm install -g @anthropic-ai/claude-code`)
- Python 3.8+ (for hook scripts)
- Git

### Optional (Recommended)
- [QMD](https://github.com/tobi/qmd): `npm install -g @tobilu/qmd`
- [GitHub CLI](https://cli.github.com/) for `/peer-scan`
- Slack MCP configured in Claude Code for `/incident-capture`, `/slack-scan`

### Environment Variables / API Keys
- **`ANTHROPIC_API_KEY`** — required by Claude Code (set in shell environment)
- **Slack MCP credentials** — configured in Claude Code's MCP settings (not in this repo); needed only for Slack subagents
- No `.env` file in this repo — all credentials are external to the vault

### Install & Start
```bash
# 1. Clone (or use as GitHub template)
git clone https://github.com/breferrari/obsidian-mind.git ~/my-vault
cd ~/my-vault

# 2. Open in Obsidian
# File → Open Vault → select ~/my-vault
# Enable Obsidian CLI: Settings → General → enable CLI (requires Obsidian 1.12+)

# 3. (Optional) Set up QMD semantic search
npm install -g @tobilu/qmd
qmd collection add . --name vault --mask "**/*.md"
qmd context add qmd://vault "Engineer's work vault: projects, decisions, incidents, people, reviews"
qmd update && qmd embed

# 4. Set up MEMORY.md pointer (one-time)
# Find your project memory path:
ENCODED=$(python3 -c "import urllib.parse; print(urllib.parse.quote('$(pwd)', safe=''))")
mkdir -p ~/.claude/projects/$ENCODED/memory
cp .claude/memory-template.md ~/.claude/projects/$ENCODED/memory/MEMORY.md

# 5. Fill in your North Star
# Edit brain/North Star.md with your goals

# 6. Start Claude Code
claude
# Claude auto-loads MEMORY.md and runs SessionStart hook → context injected

# 7. First commands
/standup   # Morning kickoff
/dump      # Brain dump anything from your day
```

### Customize
- **Your goals**: `brain/North Star.md`
- **Slash commands**: `.claude/commands/` — edit GitHub org, Slack workspace, review framework
- **Operating manual**: `CLAUDE.md` — add domain-specific conventions as you go
- **New subagents**: add `.md` files to `.claude/agents/` with YAML frontmatter
