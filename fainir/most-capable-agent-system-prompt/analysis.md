# Repository: fainir/most-capable-agent-system-prompt

**URL:** https://github.com/fainir/most-capable-agent-system-prompt

## 1. What It Does

This repo is a single, **3,228-line system prompt** (inside `README.md`) designed to be pasted into any coding agent — Claude Code, Codex, Cursor, OpenClaw, OpenCode, OpenHands, or similar — and immediately instruct that agent to **build a maximally capable, self-improving agentic operating system** from whatever environment it finds itself in. No code scaffolding is provided; the entire artifact is the prompt itself, plus an architecture SVG.

The prompt's stated ambition: create a durable platform that can handle software engineering, browser automation, desktop automation, scientific research, company-running operations, and complex multi-month projects — all from a single paste.

## 2. AI/Agent Architecture

### Core Design Philosophy: Systems Engineering Over Clever Prompting

The prompt embeds a strong, explicit philosophy that "most capable" is not about benchmark scores. It defines capability across nine orthogonal dimensions:
- **Breadth** — task type coverage
- **Depth** — long, ambiguous, multi-step work
- **Reliability** — correct completion, not just attempt
- **Transfer** — new domain adaptation
- **Memory** — knowledge preservation across sessions
- **Self-improvement** — getting better without human hand-editing
- **Governance** — knowing when NOT to act
- **Economics** — cost-aware model and method selection
- **Durability** — surviving crashes, model swaps, runtime changes

### 12-Layer Architecture (A through L)

The prompt prescribes a layered system explicitly named A–L:

| Layer | Name | Purpose |
|---|---|---|
| A | Control Plane | REST + WebSocket hub for humans: task queue, approvals, audit, cost, dashboards |
| B | Execution Fabric | Pull-based workers; claim → execute → stream → recover |
| C | Task Graph Engine | Goals → explicit tasks with typed schemas, deps, DoD, evidence |
| D | Skill & Profile System | Loadable behavior packs; ~17 named profiles (planner, tester, reviewer, etc.) |
| E | Memory System | 8 memory types: hot/warm/cold/episodic/semantic/procedural/preference/temporal |
| F | Tool Adapters | ~14 capability categories abstracted behind stable interfaces |
| G | Model Routing & Economics | Cheap→strong model ladder by task type; budget at 4 granularities |
| H | Governance, Policy & Trust | 4 autonomy levels; trust earned from outcomes, not declared |
| I | Evaluation & Learning Engine | Multi-category evals: capability, regression, behavioral, adversarial, long-horizon |
| J | Self-Improvement Engine | Inline (post-task) + background (one-change-eval-loop) modes |
| K | Observability & Incidents | Spans, traces, incidents, postmortems, stuck-task detection |
| L | Context Management | File-based state, handoffs, summaries, bounded task contexts |

### Key Agent Design Patterns

**1. Filesystem-First Project Operating System**
The canonical project state lives in files, not chat:
```
project.md, plan.md, tasks.md, knowledge.md, decisions.md,
status.md, handoff.md, FAILURE.md, artifacts/, evals/, runs/
```
Any compatible agent entering the folder should be able to continue work. Chat is disposable; files are required.

**2. Pull-Based Task Claiming**
Workers poll a queue every ~30s, atomically claim tasks, execute, verify, and report. This replaces brittle push orchestration and survives partial failures.

**3. Capability Acquisition Ladder (10 rungs)**
The system becomes capable by climbing this sequence:
1. Solve once → 2. Make repeatable → 3. Turn into a skill → 4. Turn into a workflow → 5. Turn into a specialized harness → 6. Add eval coverage → 7. Add automation → 8. Add monitoring → 9. Add trust-based autonomy → 10. Package the gain

**4. Specialized Harness Library**
9 named harness archetypes (General Dynamic, Coding & Delivery, Browser Research, Document & Contract, Finance & Reporting, Customer & Operations, Incident & Recovery, Science & Experiment, Complex Project/Company). Each harness is defined as a state machine with: trigger conditions, fixed/dynamic phases, typed inputs, validation per phase, approval gates, retry logic, stop conditions, memory updates, and its own evals.

**5. Momentum Engine ("5 queues")**
The system must always maintain: `now`, `next`, `blocked`, `improve`, `recurring`. Ending a run with all queues undefined = failure.

**6. Self-Improvement Loop (bounded, eval-gated)**
Background mode: one hypothesis → one change → delta eval → keep if better → revert if worse. Never bulk prompt surgery without eval protection. Equal-score tie → prefer simpler.

**7. Verification-First Completion**
Nothing is complete because the agent says so. Every task must define: expected output/state change, verification method, evidence to save, what failure looks like.

**8. External Intelligence Loop**
A scheduled scan of: open-source agent repos, model provider releases, protocol changes, benchmarks, research papers. Produces: digest, ranked improvement ideas, new eval candidates. Has explicit de-prioritization rules (ignore thin wrappers, generic chat shells, UI-only products).

**9. Shadow Mode & Safe Ramp-Up**
For high-risk domains: observation → recommendation → draft-with-approval → bounded autonomy. Never jump from no validation to full autonomy.

**10. Gap Classification Taxonomy**
When the system fails, it must classify into one of 13 gap types (missing skill/tool/permission/memory, bad decomposition, bad verification, unsafe autonomy, poor model routing, context overload, weak observability, missing eval, external dependency failure, bad human requirements) and choose the most leverageful repair.

### Prompt Structure Techniques

The prompt itself is a masterclass in instructing LLMs to resist common failure modes:

- **READER CONTRACT**: Explicit reading order (7 sections to read first), then create a local operating summary, then ask minimum questions, then bias toward closed loop.
- **Anti-drift rules**: "If you drift into chat-only behavior, stop and return to files." Built into the prompt as explicit stopping checks.
- **Operational over aspirational**: "A long essay about architecture without real artifact creation is failure."
- **STOPPING RULES section**: The agent should not stop after planning unless explicitly told to. It should build until milestone is verified.
- **NON-NEGOTIABLE RULES**: 10 explicit preference declarations (transparent files over hidden context, measurable outcomes over self-reported success, etc.)
- **BUILD ORDER**: Numbered 1–15 build sequence to prevent scope creep.
- **INITIAL ACTIONS**: Numbered 1–12 immediate actions the agent must take when receiving the prompt.

### Multi-Agent Patterns (7 coordination modes)

- Solo execution (simple tasks)
- Planner → executor (medium tasks)
- Planner + generator + evaluator (higher-stakes)
- Generator vs. adversarial reviewer (risky changes)
- Parallel workers (independent subtasks)
- Coordinator + specialists (broad goals)
- Proactive background agents (monitoring/improvement)

### Fractal Interface Model

The UX scales by scope, not by switching products. Same primitives at every level: ask / state / plan / tasks / artifacts / timeline / evidence / cost / approvals / memory / control. Covers 6 scope levels: micro → task → goal → project → company → portfolio.

## 3. Key Design Decisions

**Decision 1: Single agent baseline before multi-agent**
Explicit: "Do not default to a swarm of agents talking to each other." Most systems begin with one strong agent + explicit workflows, then add multi-agent only where it clearly outperforms simpler control flow.

**Decision 2: Runtime-agnostic, architecture-specific**
Don't assume product name. Build a capability matrix first (22 capability axes, 4 answers each: yes/no/partial/scored). Adapt based on what's actually available. But the architecture (task graphs, verifiers, memory layers, control plane) is never vague.

**Decision 3: Two valid implementation paths**
Harness-wrapper mode (wrap existing strong runtime) vs. native runtime mode (build on SDK). Both must maintain the same file-based project operating system contracts.

**Decision 4: SQLite first, Postgres only when scale demands**
Explicitly stated for the control plane. "SQLite is operationally simpler and surprisingly strong for the early and middle stages."

**Decision 5: Git worktrees for parallel coding isolation**
When git is available: one worktree per parallel coding task. Shared working tree only for serialized work.

**Decision 6: Typed task schema**
Tasks carry: id, goal_id, project_id, description, skill_tags, status, depends_on, owner, reviewer, priority, risk_level, budget_limit, tokens_used, attempts, verification_plan, evidence, artifacts, escalation_reason, timestamps.

**Decision 7: Approval before dispatch, not after execution**
"Risky actions should pause before side effects, not after them."

**Decision 8: Retry with variation**
"Retry with variation is better than repeat-the-same-command retries."

**Decision 9: 30-minute default task timeout, depth limit of 5**
Hard ceilings for agent recursion and task duration to prevent coordination theater.

**Decision 10: Per-skill trust, not global autonomy switch**
"A system that is good at testing is not automatically good at deploys, finance, or customer communication."

## 4. Project Structure

```
most-capable-agent-system-prompt/
├── README.md                                  # The entire artifact — 3,228-line system prompt + research appendix
└── most_capable_agent_system_architecture.svg # Architecture diagram
```

Two files total. The repo IS the prompt. No code, no dependencies, no configuration.

The README is structured as:
1. Quick start (3 lines)
2. Architecture SVG embed
3. **The Prompt** (lines 15–2965, in a fenced code block)
4. Research Appendix (annotated reference list, ~55 repos/sources)
5. Subsystem Reference Map (categorized repo lists by subsystem)
6. Notes

## 5. Tech Stack

**Runtime (proposed by the prompt, not implemented here):**
- Control plane: SQLite (WAL mode), REST + WebSocket
- Task queue: durable store (explicit goal→task graph→assignment→result lifecycle)
- Memory: layered (hot/warm/cold), structured markdown + optional graph index
- Workers: pull-based daemons, 30s polling
- Browser automation: dedicated reliability stack (named actions, session reuse, DOM evidence)
- Execution sandboxes: E2B, Daytona, or agent-sandbox (Kubernetes)
- Observability: Langfuse, Opik, or equivalent
- Model routing: LiteLLM gateway pattern
- Memory subsystem: Letta, Mem0, or Graphiti patterns
- Durable workflows: Temporal, Trigger.dev, or Inngest patterns

**The prompt itself:** Plain text in markdown. No dependencies. Runtime-agnostic by design.

## 6. Patterns Worth Borrowing

**1. The Capability Acquisition Ladder**
The 10-rung ladder (solve once → repeatable → skill → workflow → harness → eval → automation → monitoring → trust → package) is the most actionable meta-pattern. Every agent system should have this explicit progression. It prevents premature automation of unreliable processes.

**2. Momentum Queue System (5 queues)**
`now / next / blocked / improve / recurring` — always populated, never all undefined at end of run. This is a concrete anti-stall mechanism that any agent harness can adopt.

**3. Specialized Harness State Machine Pattern**
Instead of "generalist agent tries everything," define harnesses with: fixed/dynamic phases, typed inputs/outputs, validation gates per phase, compensating actions, approval gates. Especially valuable for compliance, finance, science workflows.

**4. Gap Classification Taxonomy (13 gap types)**
When anything fails, classify it deterministically. This converts ambiguous "it didn't work" feedback into targeted repair work. Far better than generic retry logic.

**5. One-Change Eval Loop for Self-Improvement**
Hypothesis → one bounded change → delta eval → keep/revert → log. Never bulk surgery without eval protection. Equal-score → prefer simpler. This is a rigorous and safe self-improvement pattern.

**6. External Intelligence Loop with Ingestion Filter**
Scheduled monitoring of the open-source ecosystem with explicit criteria: include only systems demonstrating durable execution, typed contracts, checkpointing, memory, eval loops, or traceability. Ignore thin wrappers, generic chat, trend demos. News-to-improvement pipeline converts findings into bounded experiments.

**7. Reliability Math (27 points)**
The "march of nines" framing (90% per-step reliability = system that fails frequently), plus 27 specific engineering conclusions about compensating actions, durable waits, quarantine queues, idempotency keys, checkpoint-per-phase, browser-specific reliability stack. This is the most practically dense section.

**8. READER CONTRACT Pattern**
The prompt instructs the model on exactly how to read the prompt: reading order, summary to create immediately, anti-skimming rules, closed-loop bias. This meta-instruction pattern reduces LLM drift and is worth stealing for any long operational prompt.

**9. Fractal Interface Model**
One UX mental model across 6 scope levels. Same primitives. User navigates altitude, not different products. Applicable to any complex agent-driven interface.

**10. Shadow Mode Ramp-Up**
Observation → recommendation → draft-with-approval → bounded autonomy. Mandatory for high-risk domains. Easy to implement as a trust level flag per domain.

## 7. Gaps, Risks, and Limitations

**Gap 1: No implementation provided**
The repo is a prompt, not a system. The entire claimed capability (12 layers, 9 harnesses, 5 memory types, etc.) must be built from scratch by whatever agent receives the prompt. There is no baseline implementation, no tests, no scaffolding. The gap between "prompt describes this" and "agent actually builds this reliably" is enormous.

**Gap 2: Context window overload risk**
The prompt is ~3,200 lines / ~125KB of text. Many agent runtimes have limited effective context windows. The READER CONTRACT acknowledges this ("Re-read that summary during long runs so this prompt does not get lost in the middle"), but does not solve the fundamental problem that the agent may lose track of instructions embedded deep in a 125KB prompt.

**Gap 3: No versioning or changelog**
The prompt references "Version prompts like code" as a design principle, but the repo itself has no versioning scheme, no CHANGELOG, no semantic version. It was updated as of March 28, 2026 references, but future divergence from its own principles is possible.

**Gap 4: Ambiguous agent identity in harness-wrapper mode**
When wrapping an existing runtime like Claude Code, the boundary between "the wrapped agent's behavior" and "this prompt's instructions" can be unclear. The prompt doesn't specify how to handle instruction conflicts, what takes precedence, or how to detect when the host runtime is deviating from the harness contract.

**Gap 5: No eval suite provided**
The prompt extensively describes what evals to build (capability, regression, behavioral, adversarial, long-horizon, production-derived). But there are no actual evals, no harness, no test cases. An agent instructed to "build evals" has no baseline to compare against.

**Gap 6: Science and browser domains are ambitious additions**
The science operating system (reproducibility, experiment lineage, replication queues) and browser reliability stack (session persistence, DOM evidence, selector healing) are described in depth but are genuinely hard infrastructure problems. Treating them as "just add a harness" may underestimate real implementation complexity.

**Gap 7: Multi-machine coordination is underspecified**
The prompt describes multi-machine same-project orchestration but provides minimal concrete protocols — only "coordinate through tasks, ownership, worktrees, or isolated branches." Race conditions, split-brain scenarios, and network partition handling are acknowledged but not addressed.

**Gap 8: No security threat model**
The prompt mentions "encrypt stored secrets," "permission enforcement," and "secret redaction," but provides no threat model for prompt injection, malicious tool outputs, or adversarial tasks. For a system that aspires to run companies and financial workflows, this is a meaningful omission.

**Gap 9: Implicit assumption of competent host agent**
The effectiveness of this entire system depends on the host agent (Claude Code, Codex, etc.) faithfully following ~125KB of instructions over long runs. There's no mechanism to verify the agent is following the meta-instructions, or to detect and recover from drift into "chat-only mode."

**Gap 10: The research appendix can age**
The 55-repo reference list is current as of March 28, 2026 but will age. Many referenced projects are in rapid flux (LangGraph, PydanticAI, Mastra, etc.). The prompt doesn't provide update guidance beyond "run the external intelligence loop" — which is circular if you haven't built the system yet.

## 8. How to Run It

There is nothing to run — this repo IS the prompt. To use it:

1. **Copy** the contents of the fenced code block in `README.md` (lines 15–2965)
2. **Paste** into your agent of choice as:
   - System prompt in Claude Code (`CLAUDE.md` or Claude Code system prompt field)
   - First message in Codex CLI
   - A `CURSOR.md` or rules file in Cursor
   - System prompt in OpenClaw, OpenCode, OpenHands, or any agent SDK
3. **The agent starts building immediately** — it will inspect your workspace, ask minimum questions, produce a capability matrix, write an implementation contract, and start scaffolding

**Expected agent output (per the INITIAL ACTIONS section):**
1. Runtime capability matrix (22 capabilities × 4 possible answers)
2. Implementation contract (mission, runtime profile, first milestone, constraints, safety posture, metrics, verification strategy)
3. Foundational artifact files (AGENTS.md, plan.md, tasks.md, knowledge.md, FAILURE.md, etc.)
4. 5 live momentum queues (now/next/blocked/improve/recurring)
5. First milestone definition
6. Immediate build start

**Variants offered** (noted at end of README):
- Version optimized for specific runtime
- Shorter "bootstrap first, improve forever" variant

The prompt works with any capable LLM-based coding agent. It is intentionally runtime-agnostic and requires no API keys, no dependencies, and no pre-existing codebase.
