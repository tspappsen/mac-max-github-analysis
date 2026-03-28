# Repository: aiming-lab/AutoResearchClaw

**URL:** https://github.com/aiming-lab/AutoResearchClaw

## 1. What It Does

AutoResearchClaw is a fully autonomous research pipeline that takes a single natural-language research topic and produces a complete conference-ready academic paper — with real literature from OpenAlex/Semantic Scholar/arXiv, hardware-aware sandbox experiments, statistical analysis, multi-agent peer review, and LaTeX export targeting NeurIPS/ICML/ICLR. The system runs a 23-stage pipeline across 8 phases (Scoping → Literature → Synthesis → Experiment Design → Execution → Analysis → Writing → Finalization) without human intervention, with self-healing experiment repair, autonomous PIVOT/REFINE decisions, and cross-run learning via a MetaClaw integration that converts failures into reusable skills. ~60,700 lines of Python, 1,823 tests, MIT license.

## 2. AI/Agent Architecture

### Pipeline Orchestration

The core architecture is a **linear state machine with decision loops**. `runner.py` iterates through 23 `Stage` enums, executing each via `executor.py` which dispatches to stage-specific functions in `stage_impls/`. The state machine (`stages.py`) defines:

- **Stage sequence**: Fixed 1→23 with `NEXT_STAGE` / `PREVIOUS_STAGE` navigation
- **Status transitions**: PENDING → RUNNING → DONE (or FAILED → RETRYING, BLOCKED_APPROVAL → APPROVED/REJECTED)
- **Gate stages** (5, 9, 20): Require human approval or `--auto-approve` bypass, with rollback targets (e.g., Stage 20 rejection → rollback to Stage 16)
- **Decision loops**: Stage 15 (RESEARCH_DECISION) can trigger PIVOT (→ Stage 8, re-hypothesize) or REFINE (→ Stage 13, re-run experiments), capped at `MAX_DECISION_PIVOTS = 2` to prevent infinite loops

This is NOT a ReAct-style agent — it's a **predetermined pipeline with conditional branching**. The LLM is the worker at each stage, not the decision-maker for what to do next (except at Stage 15).

### LLM Integration

Two client implementations, both exposing the same `.chat()` interface:

1. **`LLMClient`** (`llm/client.py`): OpenAI-compatible HTTP client using **stdlib only** (`urllib.request`). Features model fallback chains (e.g., gpt-5.2 → gpt-4.1 → gpt-4o-mini), exponential backoff with jitter, and `max_tokens` vs `max_completion_tokens` auto-detection per model family. Supports the OpenAI Responses API (`/responses`) alongside standard `/chat/completions`. Zero SDK dependency — raw HTTP.

2. **`ACPClient`** (`llm/acp_client.py`): Agent Client Protocol via `acpx` subprocess. Spawns persistent named sessions so the agent (Claude Code, Codex CLI, Gemini CLI, etc.) maintains context across all 23 stages. Handles OS argument-length limits by falling back to temp-file transport for large prompts. Automatic session reconnection on stale sessions.

Both clients are swappable — the pipeline uses duck typing and doesn't import specific client classes.

### Multi-Agent Subsystems

Three orchestrated multi-agent systems, all built on `BaseAgent` / `AgentOrchestrator` base classes:

1. **CodeAgent** (`pipeline/code_agent.py`): 5-phase code generation — (1) Blueprint planning with dependency DAG, (2) Sequential file generation following dependency order, (3) Execution-in-the-loop sandbox repair, (4) Solution tree search (optional), (5) Multi-agent coder-reviewer dialog. Includes AST-based hard validation gates that block identical ablations and hardcoded metrics.

2. **BenchmarkAgent** (`agents/benchmark_agent/`): 4-agent pipeline — Surveyor (searches HuggingFace/web) → Selector (tier-filters by GPU/time budget) → Acquirer (generates data loader + baseline code) → Validator (AST + import checks). Retry loop on validation failure.

3. **FigureAgent** (`agents/figure_agent/`): Hybrid pipeline — Decision Agent analyzes paper to classify figures as code-driven (charts) or conceptual (architecture diagrams). Code figures go through Planner → CodeGen → Renderer → Critic (with retry loops). Conceptual figures route to **Nano Banana** (Gemini image generation). Integrator merges all into a manifest.

The base classes provide: structural-typed LLM protocol (no import dependency), 3-tier JSON parsing (direct → fenced code block → balanced brace matching), and per-agent token tracking with counter reset to prevent double-counting on retries.

### Prompt Management

`prompts.default.yaml` defines all prompts per-stage with template variables (`{topic}`, `{exp_plan}`, etc.). The system supports user-customizable prompt files via `prompts.custom_file`. Key prompt engineering patterns:

- **Reusable blocks**: `compute_budget`, `pkg_hint_sandbox`, `topic_constraint` are injected across multiple stages
- **Anti-fabrication instructions**: Code generation prompts explicitly ban `random.uniform()` result faking, hardcoded metrics, and missing convergence checks
- **Peer review prompts**: Methodology-evidence consistency checking with specific flags for trial count fabrication, statistical test claims, and paper length

### State & Memory

- **EvolutionStore** (`evolution.py`): JSONL-backed lesson store with 30-day half-life time-decay weighting. Lessons extracted from failed stages, PIVOT/REFINE decisions, runtime warnings, and NaN/Inf metrics. `build_overlay()` generates per-stage prompt injection text.
- **MemoryStore** (`memory/store.py`): JSONL-backed memory with 3 categories (ideation, experiment, writing), confidence scoring, access tracking, and capacity-limited pruning. Supports vector embeddings for semantic retrieval (embedding field exists, but retriever uses cosine similarity over stored vectors).
- **Knowledge Base** (`knowledge/`): Per-run structured KB across 6 categories written as markdown files.
- **Checkpointing**: Atomic checkpoint writes (temp file + rename) enable `--resume` with auto-detection of last completed stage.

### Anti-Fabrication System

The **VerifiedRegistry** (`pipeline/verified_registry.py`) is the ground-truth enforcement mechanism. It builds a whitelist of all numeric values from experiment data (including rounded variants, percentage conversions, pairwise differences) and their provenance. The `paper_verifier.py` checks every number in the generated paper against this registry. Unverified numbers get sanitized. The **ExperimentDiagnosis** agent (`experiment_diagnosis.py`) classifies 13 failure modes (GPU OOM, missing deps, time guard, synthetic data fallback, identical conditions, near-random accuracy, etc.) and generates targeted repair prompts.

### MetaClaw Bridge

Optional cross-run learning system. Pipeline failures → `LessonEntry` objects → LLM converts to MetaClaw skills (SKILL.md files in `~/.metaclaw/skills/arc-*/`) → injected into future run prompts via `build_overlay()`. Includes a PRM (Process Reward Model) quality gate that uses LLM-as-judge with majority voting at gate stages.

## 3. Key Design Decisions

**Zero SDK dependency for LLM calls.** The core `LLMClient` uses only `urllib.request` — no `openai`, no `anthropic`, no `httpx` for the primary path. This makes the project extremely portable and avoids version conflicts, but means no streaming, no structured outputs, no function calling. The Anthropic adapter is loaded conditionally only when `provider: "anthropic"`.

**Pipeline over agent.** The 23-stage linear pipeline with gates is a deliberate choice over freeform agent loops. This provides reproducibility (same stages always execute), debuggability (each stage writes artifacts to `stage-XX/` directories), and cost control (bounded LLM calls). The PIVOT/REFINE loop at Stage 15 is the only dynamic routing, and it's capped at 2 pivots.

**Immutable experiment harness.** Sandbox execution wraps generated code in an immutable harness (`harness_template.py`) that captures stdout, stderr, metrics, timing, and exit code. The generated code can't bypass the harness. Combined with AST validation (no `subprocess`, `os.system`, `eval`, `exec`, `socket`), this creates a reasonable security boundary.

**ACP for agent-as-LLM.** The ACP client turns any CLI agent (Claude Code, Codex, Gemini CLI) into an LLM backend by maintaining a persistent named session via `acpx`. This means the agent accumulates context across all 23 stages — a fundamentally different interaction pattern than stateless API calls. The session reconnection logic handles the reality that CLI agents die on long-running pipelines.

**Multi-tier JSON parsing everywhere.** The `BaseAgent._parse_json()` uses 3 strategies: direct parse → fenced code block extraction → balanced brace matching. This is critical because LLMs wrap JSON in markdown, add commentary, or produce malformed output. The balanced-brace approach (counting `{` and `}`) is more robust than regex for nested structures.

**Experiment mode continuum.** Four execution modes (`simulated` → `sandbox` → `docker` → `ssh_remote`) provide a spectrum from fast/fake to real/expensive. The Docker mode supports network policies (`none`, `setup_only`, `pip_only`, `full`), auto-dependency detection from imports, and GPU passthrough. The SSH mode even supports Docker-over-SSH for maximum isolation on remote GPU servers.

**Conference template awareness.** LaTeX export targets specific conferences (`neurips_2025`, `iclr_2026`, `icml_2026`) with template-specific formatting rules, section length targets (Introduction: 800-1000 words, Method: 1000-1500 words), and a NeurIPS checklist generator. The peer review prompt literally asks "Would you recommend this paper be presented at NeurIPS/ICML?"

## 4. Project Structure

```
AutoResearchClaw/
├── researchclaw/                     # Main package (~60,700 LOC)
│   ├── cli.py                        # CLI entry point (researchclaw command)
│   ├── config.py                     # RCConfig dataclass hierarchy
│   ├── pipeline/
│   │   ├── runner.py                 # Pipeline loop, checkpoint, resume
│   │   ├── executor.py               # Stage dispatch + gate logic
│   │   ├── stages.py                 # 23-stage state machine
│   │   ├── contracts.py              # Stage I/O contracts (input/output files)
│   │   ├── stage_impls/              # Per-stage implementation functions
│   │   │   ├── _topic.py             # Stages 1-2 (scoping)
│   │   │   ├── _literature.py        # Stages 3-6 (literature)
│   │   │   ├── _synthesis.py         # Stages 7-8 (synthesis)
│   │   │   ├── _experiment_design.py # Stage 9
│   │   │   ├── _code_generation.py   # Stage 10
│   │   │   ├── _execution.py         # Stages 11-13 (experiment run)
│   │   │   ├── _analysis.py          # Stages 14-15 (analysis + decision)
│   │   │   ├── _paper_writing.py     # Stages 16-17 (outline + draft)
│   │   │   └── _review_publish.py    # Stages 18-23 (review → export)
│   │   ├── code_agent.py             # Multi-phase code generation agent
│   │   ├── verified_registry.py      # Anti-fabrication numeric whitelist
│   │   ├── experiment_diagnosis.py   # 13-class failure classifier
│   │   ├── experiment_repair.py      # Auto-repair prompt builder
│   │   ├── paper_verifier.py         # Paper ↔ experiment data checker
│   │   └── opencode_bridge.py        # OpenCode Beast Mode integration
│   ├── agents/
│   │   ├── base.py                   # BaseAgent + AgentOrchestrator
│   │   ├── benchmark_agent/          # 4-agent benchmark selection
│   │   │   ├── surveyor.py           # Search HuggingFace + web
│   │   │   ├── selector.py           # Tier-filter by GPU/time
│   │   │   ├── acquirer.py           # Generate loader/baseline code
│   │   │   ├── validator.py          # AST + import validation
│   │   │   └── orchestrator.py       # Surveyor→Selector→Acquirer→Validator
│   │   ├── code_searcher/            # Semantic code search
│   │   └── figure_agent/             # 7-agent figure generation
│   │       ├── decision.py           # Code vs image classification
│   │       ├── planner.py            # Figure planning
│   │       ├── codegen.py            # Matplotlib/TikZ generation
│   │       ├── renderer.py           # Sandbox execution
│   │       ├── critic.py             # Quality review
│   │       ├── nano_banana.py        # Gemini image generation
│   │       ├── integrator.py         # Manifest assembly
│   │       └── orchestrator.py       # Full pipeline coordination
│   ├── llm/
│   │   ├── client.py                 # OpenAI-compatible HTTP client (stdlib)
│   │   ├── acp_client.py             # ACP agent bridge via acpx
│   │   └── anthropic_adapter.py      # Anthropic Messages API adapter
│   ├── literature/
│   │   ├── arxiv_client.py           # arXiv API client
│   │   ├── openalex_client.py        # OpenAlex API client
│   │   ├── semantic_scholar.py       # S2 API client
│   │   ├── search.py                 # Multi-source search orchestrator
│   │   ├── verify.py                 # 4-layer citation verification
│   │   ├── novelty.py                # Novelty scoring
│   │   └── cache.py                  # Literature result caching
│   ├── experiment/
│   │   ├── sandbox.py                # Local Python sandbox
│   │   ├── docker_sandbox.py         # Docker container execution
│   │   ├── ssh_sandbox.py            # Remote SSH execution
│   │   ├── colab_sandbox.py          # Google Colab via Drive
│   │   ├── validator.py              # AST security validation
│   │   ├── harness_template.py       # Immutable execution harness
│   │   ├── runner.py                 # Experiment execution loop
│   │   └── visualize.py              # Chart generation
│   ├── memory/
│   │   ├── store.py                  # JSONL-backed MemoryStore
│   │   ├── retriever.py              # Semantic retrieval (cosine similarity)
│   │   ├── embeddings.py             # Embedding helpers
│   │   └── decay.py                  # Time-decay weighting
│   ├── metaclaw_bridge/
│   │   ├── session.py                # MetaClaw session management
│   │   ├── lesson_to_skill.py        # Failure → reusable skill conversion
│   │   ├── prm_gate.py               # LLM-as-judge quality gate
│   │   └── skill_feedback.py         # Skill injection into prompts
│   ├── mcp/
│   │   ├── server.py                 # MCP server (run_pipeline, get_status)
│   │   ├── tools.py                  # Tool definitions
│   │   └── transport.py              # Stdio/SSE transport
│   ├── evolution.py                  # Self-learning lesson extraction
│   ├── adapters.py                   # OpenClaw bridge adapters
│   ├── templates/                    # LaTeX conference templates
│   ├── web/                          # Web search, crawling, PDF extraction
│   └── knowledge/                    # Knowledge base (markdown/graph)
├── prompts.default.yaml              # All LLM prompt templates
├── config.researchclaw.example.yaml  # Full config reference
├── sentinel.sh                       # Background quality watchdog
├── tests/                            # 1,823 tests
├── docs/                             # Multi-language docs, showcase, guides
└── pyproject.toml                    # Hatch build system, Python 3.11+
```

## 5. Tech Stack

- **Language**: Python 3.11+ (no framework — raw stdlib where possible)
- **Build**: Hatchling (`pyproject.toml`)
- **LLM Integration**: Custom stdlib HTTP client (no SDK), supports OpenAI-compatible, Anthropic, MiniMax, DeepSeek, ACP agents
- **LLM Models referenced**: GPT-4o, GPT-5.x, o3/o4-mini, Claude, DeepSeek, Qwen, MiniMax M2.5
- **Literature APIs**: OpenAlex, Semantic Scholar, arXiv (Python `arxiv` library), Google Scholar (`scholarly`), Tavily
- **Web**: `crawl4ai` (web crawling), `PyMuPDF` (PDF extraction)
- **Experiment Execution**: Local sandbox, Docker (with GPU passthrough), SSH remote, Google Colab (via Drive polling)
- **Visualization**: Matplotlib, TikZ/PGFPlots, Gemini image generation (Nano Banana)
- **Memory/Storage**: JSONL files (no database), JSON artifacts per stage
- **External Agent Protocol**: ACP via `acpx` (supports Claude Code, Codex CLI, Copilot CLI, Gemini CLI, Kimi CLI)
- **Conference Templates**: NeurIPS 2025, ICLR 2026, ICML 2026
- **Testing**: pytest (1,823 tests, mostly unit with mocks)
- **Dependencies**: Minimal core (`pyyaml`, `rich`, `arxiv`, `numpy`), optional extras for web/PDF/anthropic

## 6. Patterns Worth Borrowing

### Stage Contract System (`pipeline/contracts.py`)
Each of the 23 stages declares its input files, output files, definition of done, error code, and max retries as a frozen dataclass. The executor validates I/O contracts before and after execution. This is an excellent pattern for any multi-stage pipeline — it makes stage dependencies explicit and catches integration bugs early.

### VerifiedRegistry Anti-Fabrication (`pipeline/verified_registry.py`)
Builds a whitelist of every numeric value from experiment data (with rounding variants, percentage conversions, and pairwise differences) so the paper verifier can flag any number that doesn't trace back to real data. The `_add_variants()` method that pre-computes rounded and converted forms is clever — it accounts for the reality that LLMs round numbers. This pattern is directly reusable for any system that generates reports from data.

### ExperimentDiagnosis Failure Taxonomy (`pipeline/experiment_diagnosis.py`)
13 classified failure modes (GPU OOM, missing deps, time guard, synthetic data fallback, identical conditions, near-random accuracy, etc.) with severity levels, affected conditions, and suggested fixes. Each check is a separate function. The `to_repair_prompt()` method serializes the diagnosis into a structured format an LLM can act on. This pattern — deterministic diagnosis → structured repair prompt → LLM fix → re-execute — is a powerful self-healing loop.

### Evolution Overlay (`evolution.py`)
30-day half-life time-decay on lessons, with `build_overlay()` generating per-stage prompt text. The combination of intra-run lessons (from the current pipeline) and inter-run MetaClaw skills (from prior pipelines) gives the system two learning loops at different timescales. The `query_for_stage()` method boosts direct stage matches 2× and errors 1.5× — a simple but effective relevance scoring.

### 3-Tier JSON Parsing (`agents/base.py`)
The `_parse_json()` method tries: (1) direct `json.loads`, (2) fenced code block extraction, (3) balanced brace matching with depth tracking. The brace-matching approach (counting `{`/`}` and trying each balanced span) is more robust than regex for nested JSON. Reusable in any system that parses LLM output.

### ACP Persistent Session (`llm/acp_client.py`)
Using `acpx` to maintain a single named session across all 23 pipeline stages means the agent accumulates full context without explicit context management. The temp-file fallback for large prompts (detecting OS argument-length limits across Windows/Linux) and automatic session reconnection are production-hardened patterns.

### Multi-Agent Orchestrator Base (`agents/base.py`)
`AgentOrchestrator` with cumulative token tracking (`_accumulate()`), iteration limits, and duck-typed LLM protocol. Sub-agents return `AgentStepResult` with success/error/token counts. This is a clean, minimal base for any multi-agent system — no framework overhead, just structural typing.

## 7. Gaps, Risks, and Limitations

### No Streaming
The stdlib HTTP client doesn't support streaming. For 32K-token paper drafts, this means long blocking waits with no progress indication. The ACP client also blocks on subprocess completion.

### MCP Server is Stub
`mcp/server.py` exposes tool definitions but most handlers return stub responses ("Stub review — not yet implemented"). The `run_pipeline` handler doesn't actually start a pipeline. This is scaffolding, not production code.

### JSONL-Only Persistence
All memory, evolution, and knowledge storage uses JSONL files. There's no database, no indexing, no concurrent access protection beyond atomic checkpoint writes. At scale (many runs, many lessons), this will become a bottleneck. The `MemoryStore.get()` method does a linear scan across all categories.

### Single-Threaded Pipeline
The pipeline is strictly sequential — no parallelism within a run. Stages like literature collection (3 APIs) and experiment runs (multiple conditions) could benefit from concurrent execution. The `openclaw_bridge.use_sessions_spawn` config flag exists but isn't implemented.

### Security Surface
The sandbox uses AST-based import allowlisting and function call blocking, but it's Python-level security (no cgroup, no seccomp). The Docker mode is properly isolated, but the default `sandbox` mode runs generated code in the same Python process context. The SSH remote mode runs code on remote machines with whatever permissions the SSH user has.

### Hardcoded Prompt Engineering
While prompts are externalizable via `prompts.custom_file`, the substantial prompt engineering in `stage_impls/` functions (anti-fabrication guards, compute budget constraints, topic constraint blocks) is embedded in Python code. Modifying these requires code changes.

### Limited Error Recovery
The retry logic is stage-level only (re-run the entire stage). There's no partial retry (e.g., re-running only failed conditions in an experiment). The `MAX_DECISION_PIVOTS = 2` cap is a blunt instrument — the system can't distinguish between productive pivots and flailing.

### Experiment Result Quality
The system relies on the LLM generating correct experiment code and the sandbox executing it faithfully. There's no symbolic verification of the code's mathematical correctness. The anti-fabrication system catches fake numbers in the paper, but can't detect if the experiment code itself implements the wrong algorithm.

### No Cost Tracking
Despite tracking token usage in `AgentStepResult`, there's no aggregate cost reporting, budget limits, or cost-per-stage breakdowns. A full pipeline run with GPT-4o could cost $20-100+ in API calls, and there's no way to set a spending cap.

## 8. How to Run It

### Prerequisites
- Python 3.11+
- Docker (optional, for isolated experiments)
- LaTeX distribution (optional, for PDF compilation)
- Node.js + npm (optional, for OpenCode Beast Mode)
- An LLM API key (OpenAI, Anthropic, DeepSeek, MiniMax, etc.)

### Environment Variables
| Variable | Required | Description |
|----------|----------|-------------|
| `OPENAI_API_KEY` | Yes (for OpenAI) | OpenAI API key |
| `ANTHROPIC_API_KEY` | For Anthropic provider | Anthropic API key |
| `TAVILY_API_KEY` | Optional | Tavily web search API key |
| `S2_API_KEY` | Optional | Semantic Scholar API key (higher rate limits) |
| `GEMINI_API_KEY` | Optional | Google Gemini (for Nano Banana figure generation) |
| `MINIMAX_API_KEY` | For MiniMax provider | MiniMax API key |
| `HF_TOKEN` | Optional | HuggingFace token (for gated models/datasets) |
| `PRM_API_KEY` | Optional | MetaClaw PRM judge API key |

### Install & Run
```bash
git clone https://github.com/aiming-lab/AutoResearchClaw.git
cd AutoResearchClaw
python3 -m venv .venv && source .venv/bin/activate
pip install -e .

# Interactive setup (checks Docker, LaTeX, installs OpenCode)
researchclaw setup

# Interactive config (creates config.arc.yaml)
researchclaw init

# Or manual config
cp config.researchclaw.example.yaml config.arc.yaml
# Edit llm.base_url, llm.api_key_env, experiment.mode

# Run
export OPENAI_API_KEY="sk-..."
researchclaw run --config config.arc.yaml --topic "Your research idea" --auto-approve
```

### Test Commands
```bash
pip install -e ".[dev]"
pytest tests/ -x -q          # Full test suite (1,823 tests)
pytest tests/test_rc_runner.py -x  # Pipeline runner tests
pytest tests/test_code_agent.py -x # Code generation agent tests
```

### Output Location
```
artifacts/rc-YYYYMMDD-HHMMSS-<hash>/
├── stage-01/ through stage-23/   # Per-stage artifacts
├── deliverables/                 # Final outputs
│   ├── paper.tex                 # Conference-ready LaTeX
│   ├── references.bib            # Real BibTeX references
│   └── charts/                   # Generated figures
├── checkpoint.json               # Resume point
├── pipeline_summary.json         # Run statistics
└── evolution/                    # Extracted lessons
```
