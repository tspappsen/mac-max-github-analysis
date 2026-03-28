# Study Index

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
