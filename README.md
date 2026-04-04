# Study Index

## fainir/most-capable-agent-system-prompt

- [fainir/most-capable-agent-system-prompt](fainir/most-capable-agent-system-prompt/analysis.md) — Analysis of the `fainir/most-capable-agent-system-prompt` repo covering its 3,228-line system prompt for building a self-improving agentic OS, 12-layer architecture (A–L), capability acquisition ladder, specialized harness library, momentum queue system, one-change eval loop, external intelligence pipeline, and fractal interface model.
	- **TL;DR:** A single ~125KB system prompt — no code, just paste into any coding agent — that instructs the agent to architect and build a maximally capable self-improving agentic OS from scratch. Standout patterns: explicit 12-layer architecture (control plane, task graph, skill profiles, 8-type memory, model routing, governance, eval engine, self-improvement, observability, context management), 10-rung capability acquisition ladder, 9 specialized harness archetypes as state machines, 5-queue momentum system, one-change-eval-loop for safe self-modification, and an external intelligence loop with explicit ingestion filters.

## jayminwest/seeds

- [jayminwest/seeds](jayminwest/seeds/analysis.md) — Analysis of the `jayminwest/seeds` repo covering its git-native JSONL issue tracker, AI-agent-first CLI design, concurrency model, template system, and os-eco ecosystem integration.
	- **TL;DR:** A lightweight, git-native issue tracker (`sd` CLI) built for AI agent workflows — JSONL as the database, advisory file locks for concurrent agents, `--json` on every command, and deep integration with the overstory/mulch/canopy toolchain.

## abhi1693/openclaw-mission-control

- [abhi1693/openclaw-mission-control](abhi1693/openclaw-mission-control/analysis.md) — Analysis of the `abhi1693/openclaw-mission-control` repo covering its three-tier agent hierarchy, schema-as-agent-interface (x-llm-intent OpenAPI extensions), markdown-as-system-prompt provisioning via Jinja2, multi-scope memory, approval-gated execution, WebSocket gateway RPC, and skills marketplace.
	- **TL;DR:** A centralized control plane (FastAPI + Next.js) for managing fleets of OpenClaw agents — standout patterns include self-directed OpenAPI discovery filtered by role tag, Jinja2 markdown files as system prompts, agent-authored SOUL.md/MEMORY.md for persistent identity, structural human-in-the-loop approval circuits, and multi-scope memory (board + group + gateway filesystem).

## paperclipai/companies

- [paperclipai/companies](paperclipai/companies/analysis.md) — Analysis of the `paperclipai/companies` repo covering its agent-company catalog, architecture, workflows, skill system, and key ideas that may be useful for Max.
	- **TL;DR:** A large catalog of reusable AI company templates for Paperclip, with strong patterns for agent roles, skills, workflows, memory, and multi-agent orchestration that could inspire Max.

## zarazhangrui/follow-builders

- [zarazhangrui/follow-builders](zarazhangrui/follow-builders/analysis.md) — Analysis of the `zarazhangrui/follow-builders` repo covering its SKILL.md-as-system-prompt pattern, two-phase prepare-then-remix architecture, central GitHub Actions feed generation, plain-English customizable prompts, and conversational onboarding flow.
	- **TL;DR:** An AI agent skill that delivers daily/weekly digests of top AI builders' posts — standout patterns include SKILL.md as the complete agent behavioral spec, a strict separation of deterministic data collection (Node.js → JSON) from LLM remixing, centrally-hosted feeds with zero user API keys required, and 3-tier prompt priority (user-custom > GitHub remote > local default).

## mattpocock/skills

- [mattpocock/skills](mattpocock/skills/analysis.md) — Analysis of the `mattpocock/skills` repo covering its 15 AI agent skill definitions for Claude Code, semantic dispatch via description-as-routing, progressive disclosure token budgeting, sub-agent fan-out patterns, PreToolUse safety hooks, and meta-skill for skill creation.
	- **TL;DR:** A curated library of 15 zero-dependency SKILL.md prompt packages for AI coding agents — standout patterns include description-as-dispatcher for semantic routing, progressive disclosure (< 100 line mains with linked deep-reference files), parallel sub-agent fan-out for divergent design, PreToolUse bash hooks for blocking dangerous git ops, and a meta-skill that teaches agents to write conformant skills.

## rohitg00/ai-engineering-from-scratch

- [rohitg00/ai-engineering-from-scratch](rohitg00/ai-engineering-from-scratch/analysis.md) — Analysis of the `rohitg00/ai-engineering-from-scratch` repo covering its 260+ lesson AI curriculum, agent loop architecture, multi-agent orchestration patterns, A2A/ACP/ANP protocol gateway, and skill/prompt artifact system.
	- **TL;DR:** A 260+ lesson open-source AI engineering curriculum (math → multi-agent swarms) where every lesson produces a reusable artifact — prompt template, skill file, or agent definition — with particularly strong implementation depth in agent loops and inter-agent communication protocols.
## affaan-m/everything-claude-code

- [affaan-m/everything-claude-code](affaan-m/everything-claude-code/analysis.md) — Analysis of the `affaan-m/everything-claude-code` repo covering its 25+ subagent system, 80+ skills, 26+ hook scripts, instinct-based continuous learning, skill-comply eval framework, observer loop prevention, model routing per agent, project-scoped memory, and cross-harness support (Claude Code, Cursor, OpenCode, Codex, Kiro).
	- **TL;DR:** A complete agent harness performance system (50K+ stars) with standout patterns including confidence-scored instinct extraction from sessions, hook-enforced quality gates, config protection guardrails, LLM-classified behavioral compliance measurement, 5-layer re-entrancy guard for hook loops, and explicit per-agent model routing — evolved over 10+ months of intensive daily use.

## aiming-lab/AutoResearchClaw

- [aiming-lab/AutoResearchClaw](aiming-lab/AutoResearchClaw/analysis.md) — Analysis of the `aiming-lab/AutoResearchClaw` repo covering its 23-stage linear research pipeline, three multi-agent subsystems (CodeAgent, BenchmarkAgent, FigureAgent), anti-fabrication VerifiedRegistry, dual learning loops (EvolutionStore + MetaClaw skills), and ACP-based CLI agent backends. 
- **TL;DR:** A ~60,700 LOC Python system that autonomously writes full ML research  standout patterns include a whitelisted numeric registry for anti-hallucination, 13-class experiment failure taxonomy with auto-repair, time-decayed evolution store for intra-run learning, and failure-to-SKILL.md conversion for inter-run improvement.papers

## oh-my-mermaid/oh-my-mermaid

- [oh-my-mermaid/oh-my-mermaid](oh-my-mermaid/oh-my-mermaid/analysis.md) — Analysis of the `oh-my-mermaid/oh-my-mermaid` repo covering its skill-as-prompt agent pattern, recursive perspective-based architecture scanner, filesystem-as-data-model storage, CLI-as-AI-output-materializer design, 7-field structured element schema, and multi-platform AI tool integration.
	- **TL;DR:** A zero-LLM-dependency CLI tool that turns AI-coded black-box codebases into navigable Mermaid architecture docs — standout patterns include shipping agentic workflows as markdown skill files that register into any AI coding tool, the AI calling deterministic shell commands (`omm write`) to materialize its analysis, and a recursive drill-down prompt that produces deep nested architecture trees in one pass.

## breferrari/obsidian-mind

- [breferrari/obsidian-mind](breferrari/obsidian-mind/analysis.md) — Analysis of the `breferrari/obsidian-mind` repo covering its Claude Code configuration-as-external-brain pattern, five lifecycle hooks (session start context injection, regex message classification, post-write frontmatter validation, pre-compact transcript backup, stop checklist), 9 subagents with YAML frontmatter declarations, 15 slash commands, vault-first memory with git-tracked markdown, and wikilink-based knowledge graph.
	- **TL;DR:** Not a software project — it's a complete Claude Code *configuration system* that turns an Obsidian vault into Claude's persistent external brain. Standout patterns include hook-driven warm context injection at session start, per-message regex classification emitting `additionalContext` routing hints, post-write schema validation for frontmatter and wikilinks, two-mode agent design (classify-then-execute), and `description` frontmatter for progressive disclosure.

## AnganSamadder/opentmux

- [AnganSamadder/opentmux](AnganSamadder/opentmux/analysis.md) — Analysis of the `AnganSamadder/opentmux` repo covering its event-driven tmux pane orchestration for OpenCode agents, session lifecycle hooks, multi-port agent support, and graceful degradation patterns.
	- **TL;DR:** An agent-agnostic OpenCode plugin that auto-spawns tmux panes per sub-agent for live execution visualization — no LLM calls, purely a DX/observability layer with clean event subscription patterns and multi-port parallel agent support.
