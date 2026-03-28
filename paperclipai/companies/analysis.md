# paperclipai/companies — Comprehensive Analysis

**Repo:** https://github.com/paperclipai/companies  
**Local clone:** `/Users/erik/opensource/companies`  
**Analyzed:** 2025  
**Scale:** 16 pre-built companies, 440+ agents, 500+ skills, ~2963 git objects

---

## What Is This Project?

`paperclipai/companies` is a **curated catalog of ready-to-deploy AI agent company templates** for the [Paperclip](https://github.com/paperclipai/paperclip) platform. Each entry in the catalog is a fully-specified "AI company" — a hierarchical organization of AI agents with defined roles, reporting structures, workflow pipelines, and reusable skill libraries.

The project functions as a **package registry / starter-kit collection** for the [Agent Companies](https://agentcompanies.io/specification) open specification, analogous to what npm or PyPI are to code packages — but instead of libraries, the packages are *organizations of AI agents*.

**Key pitch:** "Deploy an entire AI workforce in minutes." A user can take one of the 16 companies, run a single import command, and immediately have a fully configured multi-agent system operational — no design work required.

---

## What Does It Do?

The repo provides 16 distinct "companies," each targeting a different domain:

| Company | Domain | Agents | Skills |
|---|---|---|---|
| **GStack** | Software engineering (Garry Tan's workflow) | 5 | 27 |
| **Superpowers Dev Shop** | Disciplined TDD software development | 4 | 14 |
| **Agency Agents** | Full-spectrum AI agency (167 agents, 10 divisions) | 167 | — |
| **Aeon Intelligence** | Autonomous research/crypto/engineering on GitHub Actions | 4 | 32 |
| **AgentSys Engineering** | End-to-end dev pipeline with slop detection | 5 | 14 |
| **ClawTeam Capital** | Multi-perspective investment analysis | 7 | 1 |
| **ClawTeam Engineering** | Self-organizing software dev teams | 5 | 1 |
| **ClawTeam Research Lab** | ML research automation | 4 | 1 |
| **Donchitos Game Studio** | Full-service indie game studio (10 dev phases) | 48 | 38 |
| **Fullstack Forge** | Software consultancy (66 skills, 12 languages) | 49 | 66 |
| **K-Dense Science Lab** | Multi-disciplinary science institute (177 skills) | 54 | 177 |
| **MiniMax Studio** | Digital studio (apps, shaders, documents) | 5 | 10 |
| **Product Compass Consulting** | AI product management (65 PM frameworks) | 48 | 65 |
| **RedOak Review** | Code/design/security review agency | 5 | 6 |
| **TÂCHES Creative** | Creative strategy & meta-skills agency | 6 | 35 |
| **Trail of Bits Security** | Security auditing & verification firm | 28 | 35 |

Additionally the repo contains two shared meta-skills in `skills/`:
- **company-creator** — a skill for creating new agent companies from scratch or from an existing repo
- **readme-updater** — a skill for updating the root README.md catalog

---

## How Does It Work? (Architecture & Data Flow)

### The Agent Companies Specification

The whole system is built on the [Agent Companies v1 specification](https://agentcompanies.io/specification). This is an open standard for packaging agent organizations as portable, importable artifacts. The spec defines:

**File types and their roles:**

| File | Kind | Purpose |
|---|---|---|
| `COMPANY.md` | company | Root entrypoint — org boundary, defaults, metadata |
| `AGENTS.md` | agent | One role — instructions, skills, reporting |
| `SKILL.md` | skill | Reusable capability package |
| `TEAM.md` | team | Reusable org subtree (optional grouping) |
| `PROJECT.md` | project | Planned work grouping (optional) |
| `TASK.md` | task | Portable starter task with scheduling (optional) |
| `.paperclip.yaml` | vendor extension | Paperclip-specific: adapter types, env secrets |

**Directory layout for every company:**
```
<company-slug>/
├── COMPANY.md           # Required: metadata, goals, schema
├── agents/
│   └── <slug>/AGENTS.md # One directory per agent
├── skills/
│   └── <slug>/SKILL.md  # One directory per skill
├── teams/
│   └── <slug>/TEAM.md   # Optional team groupings
├── images/
│   └── org-chart.png    # Visual org chart
└── .paperclip.yaml      # Paperclip vendor extension
```

### Frontmatter-Based Metadata

Every file uses YAML frontmatter for machine-readable metadata:

```yaml
# COMPANY.md frontmatter
schema: agentcompanies/v1
name: Trail of Bits Security
slug: trail-of-bits-security
version: 1.0.0
license: CC-BY-SA-4.0
authors:
  - name: Trail of Bits
goals:
  - Conduct rigorous security audits...
metadata:
  sources:
    - kind: github-dir
      repo: trailofbits/skills
      path: .
      commit: 5c15f4f5644b4bd3d48882a802a7232d501852b6
      usage: referenced
```

```yaml
# AGENTS.md frontmatter
name: Smart Contract Auditor
title: Senior Smart Contract Auditor
reportsTo: blockchain-security-lead
skills:
  - building-secure-contracts
```

```yaml
# SKILL.md frontmatter (referenced external skill)
name: property-based-testing
description: Property-based testing guidance for multiple languages
metadata:
  sources:
    - kind: github-file
      repo: trailofbits/skills
      path: plugins/property-based-testing/skills/property-based-testing/SKILL.md
      commit: 5c15f4f5644b4bd3d48882a802a7232d501852b6
      usage: referenced
```

### The Reference/Vendor Pattern

Skills can be either **vendored** (content copied inline) or **referenced** (pointer to upstream commit SHA). The default and strongly preferred approach is **referenced** — a SKILL.md file contains just frontmatter pointing to the external source with a pinned git commit SHA. This keeps the companies repo lean (just pointers) while the actual skill logic lives in the upstream repos.

```yaml
metadata:
  sources:
    - kind: github-file
      repo: obra/superpowers
      path: skills/test-driven-development/SKILL.md
      commit: 1128a721ca3b7fd76bf12e8392cdeb89cfcfcf2a
      attribution: Jesse Vincent
      license: MIT
      usage: referenced
```

### Workflow Patterns

Companies implement one of four workflow patterns, explicitly documented:

1. **Pipeline** — sequential stages, each agent hands off to the next (GStack: CEO → CTO → Staff Engineer → Release Engineer → QA)
2. **Hub-and-spoke** — manager dispatches to specialists who report back (Aeon: CIO synthesizes results from Research Analyst, Engineering Lead, Crypto Analyst)
3. **Collaborative** — agents work together as peers (TÂCHES Creative)
4. **On-demand** — specialists summoned as needed (Agency Agents, Fullstack Forge)

### Organizational Hierarchy

Every company must form a valid directed tree:
- Every agent has a `reportsTo` field pointing to their manager's slug
- The CEO has `reportsTo: null`
- Teams group agents with a `manager` field and `includes` list
- The spec requires every agent to ultimately report to a CEO

### .paperclip.yaml — Vendor Extension

The Paperclip platform's config file specifies:
- `adapter.type` — which AI runtime to use (`claude_local`, `codex_local`, `opencode_local`, `cursor`, `gemini_local`, `openclaw_gateway`)
- `inputs.env` — secrets/env vars needed by specific agents (e.g., `GH_TOKEN`, `ALCHEMY_API_KEY`)

```yaml
schema: paperclip/v1
agents:
  release-engineer:
    adapter:
      type: claude_local
      config:
        model: claude-sonnet-4-6
    inputs:
      env:
        GH_TOKEN:
          kind: secret
          requirement: optional
```

### Import Mechanism

Companies are imported via the Paperclip CLI:
```bash
npx paperclipai company import --from ./trail-of-bits-security
# or from GitHub directly:
npx companies.sh add paperclipai/companies/trail-of-bits-security
```

---

## How Do You Run It? (Setup & Usage)

### Prerequisites
- [Paperclip](https://github.com/paperclipai/paperclip) platform installed
- Appropriate AI runtime CLI (e.g., Claude Code CLI for `claude_local` adapter)
- Any API keys required by specific company skills (e.g., `GH_TOKEN`, `ALCHEMY_API_KEY`)

### Import a Company
```bash
# Clone the repo
git clone https://github.com/paperclipai/companies

# Import a company into Paperclip
npx paperclipai company import --from ./trail-of-bits-security

# Or use the shorthand
npx companies.sh add paperclipai/companies/trail-of-bits-security
```

### Create a New Company (using the company-creator skill)
```bash
# Install the company-creator skill
npx skills install paperclipai/companies/skills/company-creator

# Then invoke it with any agent
"Use the company-creator skill to create a new company from https://github.com/some/repo"
```

The company-creator skill then:
1. Analyzes the repo (clones, reads README, scans for existing skills/agents)
2. Interviews the user (proposed hiring plan, workflow pattern, naming)
3. Reads the spec
4. Generates all files
5. Asks where to write the output
6. Writes README.md and LICENSE

### Dry-Run Testing
```bash
paperclipai company import --from /path/to/your-company --dry-run
```

---

## Key Features and Capabilities

### 1. Complete Org Structures with Reporting Lines
Every company ships with a full org chart — not just a list of agents, but a defined hierarchy with explicit `reportsTo` relationships. Work flows through the org based on these lines, with each agent knowing exactly who gives them work and who they hand off to.

### 2. Workflow-Aware Agent Instructions
Each AGENTS.md body is structured around four questions:
- **Where work comes from** (what triggers this agent)
- **What it does** (the core behavior)
- **What it produces** (the output artifact)
- **Who it hands off to** (the next agent in the chain)

This produces agents that operate as organizational members, not isolated tools.

### 3. Skill Reuse via Pinned References
The reference system allows skills to live in their canonical upstream repos while being incorporated into multiple companies. A skills author publishes one SKILL.md; companies reference it by commit SHA. This enables independent versioning and eliminates duplication.

### 4. Multi-Model Orchestration
Some companies specify different AI models per tier:
- Donchitos Game Studio uses Opus-tier for CEO and Directors, Sonnet-tier for Department Leads, Haiku-tier for Specialists
- GStack's codex skill uses OpenAI Codex as a "second opinion" cross-model reviewer
- AgentSys's learn skill creates RAG-optimized knowledge indexes

### 5. Scheduled Autonomous Operation
Aeon Intelligence demonstrates fully autonomous operation via GitHub Actions cron schedules:
- Daily morning briefs (synthesizes overnight activity)
- Weekly reviews (goal progress, self-reflection)
- Heartbeat health checks
- Memory flush and consolidation

### 6. Shared Memory System
The default company template includes a three-layer PARA memory system:
- **Knowledge graph** — entities and facts
- **Daily notes** — activity logs
- **Tacit knowledge** — synthesized patterns
- Agents write to `AGENT_HOME/MEMORY.md` for continuity across runs

### 7. Parallel Agent Dispatch
Superpowers' `dispatching-parallel-agents` skill and ClawTeam's worktree isolation enable concurrent subagent execution for independent tasks without state conflicts.

### 8. Anti-Slop Mechanisms
AgentSys includes `deslop` — a skill specifically for detecting and cleaning AI-generated code slop (verbose comments, unnecessary boilerplate, over-engineered abstractions) using certainty-based findings and auto-fixes.

### 9. Drift Detection
AgentSys's `drift-analysis` skill detects when implementations have drifted from their plans and creates prioritized reconstruction plans.

---

## Notable Patterns, Algorithms, and Techniques

### Phase-Gated Pipelines
AgentSys and GStack enforce strict phase gates — no agent can proceed without the prior phase completing:
```
Discovery → Exploration → Planning → Implementation → Review → Shipping
```
Each gate is documented in the CEO's workflow instructions, preventing agents from skipping steps.

### NEXUS Framework (Agency Agents)
Agency Agents implements a seven-phase delivery pipeline: Discovery → Strategy → Foundation → Build → Hardening → Launch → Operate. Quality gates at each phase prevent progression until criteria are met.

### MDA Framework (Donchitos Game Studio)
The Creative Director uses the Mechanics-Dynamics-Aesthetics framework to evaluate creative proposals, ensuring game feel is analyzed at all three levels.

### Hub-and-Spoke with Mailbox Messaging (ClawTeam)
ClawTeam companies use a mailbox system for inter-agent communication, enabling parallel independent research without interference. Agents work in isolated git worktrees.

### Meta-Prompting (TÂCHES Creative)
Separates *analysis* from *execution* by generating specification-grade prompts in a first pass, then running them in fresh sub-agent contexts. This avoids context contamination between planning and execution.

### Decision Escalation (Tâches / Multiple)
Agents are explicitly forbidden from making certain categories of decisions unilaterally. Creative Director cannot write code; CEO cannot approve individual assets. This mirrors real org governance.

### 10-11 Thinking Frameworks as Skills (TÂCHES Creative)
Mental models are codified as discrete reusable skills:
- `consider-first-principles` — break to fundamentals, rebuild from base truths
- `consider-inversion` — reason backwards from failure
- `consider-pareto` — 80/20 lens
- `consider-second-order` — downstream consequences
- `consider-occams-razor` — simplest explanation
- `consider-via-negativa` — define by what to remove
- `consider-eisenhower-matrix` — urgency/importance 2x2
- `consider-opportunity-cost` — explicit tradeoff analysis
- `consider-10-10-10` — 10 min / 10 month / 10 year consequence lens

### Cross-Model Verification
GStack's `codex` skill uses OpenAI Codex CLI as an adversarial reviewer from a different model family. Trail of Bits' `second-opinion` skill does the same with Gemini. This provides cross-model blind spot detection.

### Property-Based Testing Discipline (Trail of Bits)
Rather than just unit tests, Trail of Bits encodes property-based testing as a first-class skill — generating random inputs to find invariant violations across multiple languages and smart contract environments.

### Variant Analysis (Trail of Bits)
After finding a bug, the `variant-analysis` skill systematically searches for all similar instances in the codebase, turning a single finding into a comprehensive bug-class assessment.

---

## Data Formats and Schemas

### Agent Companies v1 Spec (YAML Frontmatter)

**COMPANY.md required fields:**
```yaml
schema: agentcompanies/v1
name: Company Name
description: What this company does
slug: company-slug
```

**COMPANY.md optional fields:**
```yaml
version: 0.1.0            # semver
license: MIT
authors:
  - name: Jane Doe
goals:
  - List of company goals
metadata:
  sources: [...]           # upstream source references
requirements:
  secrets: [...]           # required API keys
```

**AGENTS.md required fields:**
```yaml
name: Agent Name
title: Role Title
reportsTo: manager-slug   # or null for CEO
```

**AGENTS.md optional fields:**
```yaml
skills:
  - skill-shortname       # resolves to skills/<shortname>/SKILL.md
slug: agent-slug
metadata:
  sources: [...]          # when agent content is referenced
```

**SKILL.md fields:**
```yaml
name: skill-name
description: Short description
metadata:
  sources:
    - kind: github-file | github-dir
      repo: owner/repo
      path: path/to/file
      commit: full-sha
      sha256: optional-hash
      attribution: Author Name
      license: MIT
      usage: referenced | vendored | mirrored
```

**TEAM.md fields:**
```yaml
name: Team Name
description: What this team does
slug: team-slug
manager: ../../agents/slug/AGENTS.md
includes:
  - ../../agents/slug/AGENTS.md
  - ../../skills/slug/SKILL.md
tags: [engineering]
```

**TASK.md scheduling fields:**
```yaml
schedule:
  timezone: America/Chicago
  startsAt: 2026-03-16T09:00:00-05:00
  recurrence:
    frequency: weekly
    interval: 1
    weekdays: [monday]
    time:
      hour: 9
      minute: 0
```

**Paperclip vendor extension (.paperclip.yaml):**
```yaml
schema: paperclip/v1
agents:
  agent-slug:
    adapter:
      type: claude_local | codex_local | opencode_local | cursor | gemini_local | openclaw_gateway
      config:
        model: claude-sonnet-4-6
    inputs:
      env:
        API_KEY:
          kind: secret
          requirement: optional | required
```

---

## External Services and APIs Used

### AI Runtimes (via Paperclip adapters)
- **Claude Code CLI** (`claude_local`) — most common; used by Superpowers, AgentSys, Aeon, GStack, Donchitos
- **OpenAI Codex CLI** (`codex_local`) — used by GStack's `codex` skill as a cross-model reviewer
- **Google Gemini CLI** (`gemini_local`) — used by Trail of Bits' `second-opinion` skill
- **Cursor** (`cursor`) — supported adapter type
- **OpenCode CLI** (`opencode_local`) — supported adapter type
- **OpenClaw Gateway** (`openclaw_gateway`) — Paperclip's own gateway

### Version Control & Automation
- **GitHub** — primary platform; `GH_TOKEN` is the most common secret requirement
- **GitHub Actions** — Aeon Intelligence runs entirely via cron-triggered Actions workflows
- **Git worktrees** — used by ClawTeam and Superpowers for parallel agent isolation

### Communication Channels (Aeon Intelligence)
- **Telegram** — user interaction channel
- **Discord** — alternative notification channel
- **Slack** — alternative notification channel

### Domain-Specific External APIs

**Crypto / Web3 (Aeon Intelligence):**
- Alchemy API (`ALCHEMY_API_KEY`) — on-chain data

**Science (K-Dense Science Lab) — 37 databases:**
- AlphaFold Database, ArXiv, BioRxiv, BindingDB, ChEMBL, ClinicalTrials.gov, ClinVar, cBioPortal, ChemBL, BRENDA, PubMed (via biopython/bioservices)
- Alpha Vantage (financial data)

**Security (Trail of Bits):**
- Semgrep (static analysis)
- BurpSuite (web security)
- YARA (malware detection)

**Product Analytics (Product Compass):**
- SQL query skills for data warehouses

### MCP Servers
TÂCHES Creative's `create-mcp-servers` skill documents creating Model Context Protocol servers that expose tools/resources/prompts to Claude — used for integrating custom APIs into agent workflows.

---

## LLM/AI Integration Details

### Agent Runtime
The companies are designed to run agents powered by LLM CLI tools. The primary runtime is **Claude Code** (Anthropic's CLI for Claude), but the system is runtime-agnostic via the adapter system. Each agent is essentially a system prompt (the AGENTS.md body) loaded into the AI runtime, with attached skills providing additional context/instructions.

### Skill Injection
When an agent has skills listed in its frontmatter:
```yaml
skills:
  - test-driven-development
  - writing-plans
```
The Paperclip platform loads the content of `skills/test-driven-development/SKILL.md` and `skills/writing-plans/SKILL.md` into the agent's context alongside its AGENTS.md instructions. Skills are essentially modular system prompt extensions.

### Model Tiering
Donchitos Game Studio explicitly encodes a three-tier model selection strategy:
- **Opus** (most capable, most expensive) — CEO, Creative Director, Technical Director, Producer
- **Sonnet** (balanced) — Department Leads (Game Designer, Lead Programmer, Art Director)
- **Haiku** (fast, cheap) — Line specialists and workers

This mirrors real-world cost optimization for agentic workloads.

### Multi-Model Adversarial Review
Several skills use a second LLM from a different provider as an adversarial reviewer:
- GStack's `codex`: Claude implements, Codex reviews. Three modes: Review (pass/fail gate), Challenge (adversarial edge-case finder), Consult (freeform with session continuity).
- Trail of Bits' `second-opinion`: runs Codex or Gemini on uncommitted changes, branch diffs, or specific commits.
- AgentSys's `consult`: routes to multiple AI tools with different perspectives.

### Memory Architecture (Default Template)
The default agent template references a `para-memory-files` skill that implements a three-layer memory system:
- **Knowledge graph** — entities, facts with atomic fact schemas
- **Daily notes** — activity logs with PARA folder structure
- **Tacit knowledge** — synthesized patterns and weekly synthesis
- Memory decay rules built in (stale entries are pruned)
- `qmd recall` for structured memory querying

### Autonomous Scheduling (Aeon Intelligence)
Aeon Intelligence demonstrates fully autonomous LLM operation on cron schedules:
- Skills fire on GitHub Actions cron triggers
- Each run reads `MEMORY.md` first for continuity
- Outputs are written back to the repo
- Morning brief synthesizes overnight activity across all agents
- Memory-flush skill prunes stale entries weekly

### RAG Integration (AgentSys)
The `learn` skill creates "RAG-optimized indexes" from web research — structured knowledge documents optimized for retrieval by future agent runs.

### Context Handoff (TÂCHES Creative)
The `context-handoff` skill captures structured state for long-running tasks spanning multiple agent sessions:
- Original task description
- Work completed so far
- Work remaining
- Attempted approaches and their outcomes
- Critical context snippets
- Current state snapshot

This enables reliable multi-session workflows without losing progress.

---

## Relevance to Max

Max is an always-on AI assistant daemon with Telegram/TUI interfaces, worker session management, long-term memory, and a skill system. The paperclipai/companies repo is highly relevant across multiple dimensions:

### 1. Skill System Architecture — Direct Adoption Candidate

The Agent Companies skill format (`SKILL.md` with YAML frontmatter + body) is clean and well-thought-out. Max's skill system could adopt this exact format:

```yaml
---
name: my-skill
description: When to trigger this skill
metadata:
  sources:
    - kind: github-file
      repo: owner/repo
      path: path/to/SKILL.md
      commit: <sha>
      usage: referenced
---
# Skill instructions here
```

**What to borrow:**
- The `description` field as a natural-language trigger condition (not just a name) — this is how the skill tells the agent when to invoke it
- The `usage: referenced | vendored | mirrored` pattern for managing external skill sources
- Pinned commit SHAs for reproducible skill versions
- The `sources` attribution trail for tracking where skills come from

### 2. Worker/Agent Role Definitions — Workflow-Aware Prompt Pattern

The four-question AGENTS.md structure is an excellent pattern for Max's worker sessions:

```markdown
## Where work comes from
## What you do
## What you produce  
## Who you hand off to / What triggers you
```

Max's worker prompts would benefit enormously from this structure. Currently workers are likely flat instructions; this pattern gives them organizational context, making handoffs and routing explicit rather than implicit.

### 3. Memory Architecture — Adopt the PARA System

The default template's `para-memory-files` skill implements exactly the kind of memory Max needs:
- **Three layers**: knowledge graph (facts/entities) + daily notes (activity log) + tacit knowledge (synthesized patterns)
- **Atomic fact schemas** — structured JSON/YAML facts with decay rules
- **Weekly synthesis** — automated consolidation runs
- **qmd recall** — structured memory querying syntax

Max already has long-term memory; the PARA structure would organize it more effectively. The `memory-flush` and `reflect` skills from Aeon Intelligence are directly applicable:
- `memory-flush` — weekly pruning of stale entries
- `reflect` — self-reflection on recent patterns to identify improvements
- `heartbeat` — daily health check verifying all subsystems are operational

### 4. Scheduled Autonomous Operation — Aeon Intelligence Pattern

Aeon Intelligence runs entirely autonomously on cron schedules, which is exactly what Max does as an "always-on daemon." The Aeon pattern is directly applicable:

- **Morning brief**: synthesize overnight activity across all subsystems into a single Telegram message
- **Weekly review**: assess progress against goals, reflect on patterns
- **Goal tracker**: maintain and check progress against explicit goals
- **Skill-health**: verify all skills are functional, no runs were missed

Max could implement these as first-class scheduled tasks modeled directly on Aeon's skill set.

### 5. Multi-Model Adversarial Review — Cross-Model Blind Spots

GStack's `codex` skill and Trail of Bits' `second-opinion` skill demonstrate the value of using a second LLM model to review work from a different perspective. For Max, this could mean:
- Running a second model on generated content before delivering it
- Using a cheaper/faster model for initial drafts and a premium model for review
- Adversarial prompting to catch reasoning errors before output reaches the user

### 6. Context Handoff — Multi-Session Task Continuity

TÂCHES Creative's `context-handoff` skill solves the exact problem Max faces with long-running tasks:

```
Context handoff document structure:
- Original task + why it matters
- Work completed (with file-level specifics)
- Work remaining (prioritized)
- Attempted approaches + why each failed/succeeded
- Critical context (key decisions, gotchas)
- Current state (exact cursor position in the work)
```

Max should implement a formal context-handoff mechanism for tasks that span multiple worker sessions. This prevents the common failure mode where a resumed task loses the thread.

### 7. Thinking Frameworks as Composable Skills

TÂCHES Creative's consider-* skills encode mental models as reusable skills. Max could benefit from similar composable reasoning primitives:
- `consider-first-principles` — for novel or difficult requests
- `consider-inversion` — for planning and risk assessment
- `consider-second-order` — for decision-making
- `consider-opportunity-cost` — for resource allocation decisions

These could be auto-selected by Max based on request type, or explicitly invoked by the user.

### 8. Drift Detection — Plan vs. Reality Monitoring

AgentSys's `drift-analysis` skill is directly applicable to Max's long-running tasks:
- Analyze current project state against the original plan
- Detect when implementation has diverged from intent
- Create prioritized reconstruction plans

For Max, this could be a background task that runs periodically on active projects to flag when a worker session has gone off-track.

### 9. Anti-Slop Quality Gate

AgentSys's `deslop` skill — certaint-based cleaning of AI-generated code slop — is a valuable addition to Max's code-related worker sessions. Running deslop as a post-processing step before delivering code outputs would improve quality systematically.

### 10. Hub-and-Spoke Routing with ClawTeam's Mailbox Pattern

ClawTeam's mailbox system for inter-agent communication, combined with git worktree isolation for parallel work, maps well to Max's worker session model:
- Each worker session could have an isolated worktree
- Workers communicate via a message queue (Max already has Telegram)
- Parallel workers on independent tasks don't conflict

### 11. Phase-Gated Pipelines for Complex Tasks

AgentSys and GStack's phase-gate pattern (Discovery → Planning → Implementation → Review → Shipping) is a strong pattern for Max's complex, multi-step tasks. Explicit phase gates prevent Max from skipping directly to implementation without adequate planning.

### 12. The company-creator Pattern — Meta-Skill for Max Configuration

The `company-creator` skill itself is an excellent model for a "Max configurator" meta-skill — a skill that interviews the user, analyzes their workflow, and generates customized Max configuration (new skills, new worker role definitions, updated routing rules). The interview-first, propose-concrete-plan approach is a strong UX pattern.

### Summary of Highest-Value Adoptions

| Pattern | Source | Max Application |
|---|---|---|
| SKILL.md format with trigger descriptions | All companies | Standardize Max's skill format |
| Four-question worker instruction structure | All companies | Worker session prompt templates |
| PARA memory + memory-flush + reflect | Aeon / Default | Max memory organization and hygiene |
| Morning brief / weekly review schedules | Aeon | Daily/weekly autonomous Telegram summaries |
| Context handoff documents | TÂCHES | Multi-session task continuity |
| Cross-model adversarial review | GStack / Trail of Bits | Second-opinion quality gate |
| Phase-gated pipelines | AgentSys / GStack | Complex task execution guardrails |
| Drift detection | AgentSys | Background plan-vs-reality monitoring |
| Anti-slop post-processing | AgentSys | Code output quality gate |
| Thinking framework skills | TÂCHES | Composable reasoning primitives |
| Worktree isolation for parallel workers | ClawTeam / Superpowers | Parallel Max worker isolation |

---

## File Count and Scale Summary

```
Total files: ~2963 git objects
Companies: 16
Total agents: ~440+
Total skills: ~500+
Skill types: referenced (majority), vendored (minority)
Schema version: agentcompanies/v1
Primary runtime: claude_local (Claude Code CLI)
License: MIT (repo); individual companies retain upstream licenses
```

The repo is growing — it accepts PRs for new companies and has a formal CONTRIBUTING.md with conventions for slug naming, spec version, agent reporting structure, and testing imports with `--dry-run`.
