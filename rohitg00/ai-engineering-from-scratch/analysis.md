# Repository: rohitg00/ai-engineering-from-scratch

**URL:** https://github.com/rohitg00/ai-engineering-from-scratch

## 1. What It Does

This is a comprehensive, open-source AI engineering curriculum structured as 260+ hands-on lessons across 20 phases — from linear algebra fundamentals through to autonomous multi-agent swarms. It's not a framework or library; it's a *teaching codebase* where every lesson produces a reusable artifact: a prompt template, a SKILL.md file (for AI coding agents like Claude Code/Cursor), an agent definition, or an MCP server. The project solves the "breadth vs. depth" problem in AI education by covering the full stack — math, ML, deep learning, NLP, vision, speech, transformers, LLMs, RAG, agents, multi-agent systems, and production infrastructure — while ensuring each lesson results in runnable code and a shippable output. As of now, ~60 of the 260+ lessons are complete (Phases 0–2 fully done, with key lessons in Phases 3, 7, 10, 14, and 16).

## 2. AI/Agent Architecture

The most architecturally interesting content lives in Phases 13–16 (Tools & Protocols, Agent Engineering, Autonomous Systems, Multi-Agent & Swarms). Here's how the pieces connect:

### The Agent Loop (Phase 14, Lesson 1)
The core pattern is a `while` loop with an LLM inside. Both the Python (`SimpleAgent`) and TypeScript implementations define a tool registry (read_file, write_file, run_command, list_files), then loop: send messages → LLM decides tool calls or final response → execute tools → feed results back. The `callLLM` function is stubbed (no real API call) to keep the lesson framework-agnostic. The architecture is explicitly modeled on Claude Code/Cursor/Devin.

### Multi-Agent Orchestration (Phase 16)
**Lesson 1 (Why Multi-Agent)** implements three orchestration patterns in TypeScript:
- **Single-agent baseline**: one context window, one system prompt, serial execution — demonstrates context saturation
- **Pipeline pattern**: specialist agents (researcher → coder → reviewer) pass messages sequentially, each with isolated context
- **Fan-out pattern**: researcher and requirements analyst run in `Promise.all()`, results merge for the coder

Each agent is a `SpecialistAgent` with its own system prompt. Messages between agents are typed (`AgentMessage { from, to, content, timestamp }`). The lesson uses fake LLM calls to compare token usage across patterns.

### Communication Protocols (Phase 16, Lesson 3)
The most sophisticated code in the repo. A 744-line TypeScript implementation unifying four protocols:

- **MCP** (tool access) — referenced from Phase 13, agent-to-tool
- **A2A** (Agent2Agent, Google/LF) — `AgentCard` discovery via skill tags, `TaskManager` with full lifecycle (submitted → working → input-required → completed/failed/canceled), SSE-style streaming via `AsyncGenerator<TaskEvent>`
- **ACP** (Agent Communication Protocol) — `AuditableRunner` that logs every run with trajectory metadata (reasoning, tool name, input/output, timestamps), queryable by agent, session, or run ID
- **ANP** (Agent Network Protocol) — `IdentityRegistry` with DID documents, Ed25519 key pairs, signature verification, and human authorization checks

These are wired together through a `ProtocolGateway` class that: verifies identity (ANP) → discovers agents (A2A) → delegates tasks (A2A + ACP) → produces audit trails (ACP). The demo shows a coder agent delegating research to a researcher agent with full cryptographic identity verification.

### Prompt & Skill Outputs
Each lesson produces `prompt-*.md` (system prompts for specific tasks like debugging agent loops, choosing distance metrics, explaining attention) and `skill-*.md` files (structured descriptions of capabilities, designed to be loaded into AI agents via [SkillKit](https://github.com/rohitg00/skillkit)). This is a meta-pattern: the course *about* building agents also produces artifacts *for* agents.

### What's Planned But Not Built Yet
Phases 11–15 are mostly ⬚ (planned): RAG pipelines, function calling, MCP servers/clients, context window management, context compression, subagent delegation, permissions/sandboxing, file-based task systems, eval-driven development, autonomous loops, self-healing agents. The roadmap is detailed but the implementation is early.

## 3. Key Design Decisions

**"Build from scratch, then use frameworks"** — Every concept is first implemented with raw numpy/stdlib before touching PyTorch/transformers. The BPE tokenizer exists in Python, Rust, *and* as a comparison with tiktoken. Self-attention is pure numpy before any HuggingFace. This matters because it forces understanding of the primitives that frameworks hide.

**Multi-language by design** — Python for ML/DL, TypeScript for agent engineering and tool protocols, Rust for performance-critical paths (tokenizers, inference), Julia for math. The choice of TypeScript for agent work (Phases 14–16) reflects the real-world pattern of agent tooling being built in TS (Claude Code, Vercel AI SDK, LangChain.js).

**Artifacts as first-class output** — The `outputs/` directory with `prompt-*.md` and `skill-*.md` is not decoration. The project's thesis is that every lesson should produce something installable into an AI agent. The `outputs/index.json` tracks prompts, skills, agents, and MCP servers as a registry.

**Protocol-native agent design** — The communication protocols lesson doesn't just explain A2A/ACP/ANP; it implements a working `ProtocolGateway` with real Ed25519 crypto, DID documents, and task lifecycle management. This is more implementation depth than most production agent frameworks provide for inter-agent communication.

**Fake LLM calls for testability** — All agent code stubs out the LLM (`callLLM` returns canned responses). This is deliberate: it lets the architecture be tested and demonstrated without API keys or costs. Real LLM integration is deferred to "use it with a real API key" sections.

## 4. Project Structure

```
ai-engineering-from-scratch/
├── phases/                          # 20 phase directories
│   ├── 00-setup-and-tooling/        # ✅ 12 lessons: env, git, GPU, Docker
│   ├── 01-math-foundations/         # ✅ 22 lessons: linalg, calc, prob, info theory
│   ├── 02-ml-fundamentals/          # ✅ 18 lessons: regression, trees, SVM, pipelines
│   ├── 03-deep-learning-core/       # 🚧 perceptron, backprop, activations (4/13)
│   ├── 04-computer-vision/          # ⬚ CNNs, YOLO, diffusion, ViT
│   ├── 05-nlp-foundations/          # ⬚ tokenization, embeddings, attention
│   ├── 06-speech-and-audio/         # ⬚ ASR, TTS, Whisper
│   ├── 07-transformers-deep-dive/   # 🚧 self-attention from scratch (1/14)
│   ├── 08-generative-ai/           # ⬚ VAE, GANs, diffusion, flow matching
│   ├── 09-reinforcement-learning/   # ⬚ Q-learning, PPO, RLHF
│   ├── 10-llms-from-scratch/        # 🚧 BPE tokenizer in Py+Rust (1/14)
│   ├── 11-llm-engineering/          # ⬚ prompting, RAG, function calling, guardrails
│   ├── 12-multimodal-ai/           # ⬚ CLIP, multimodal RAG, multimodal agents
│   ├── 13-tools-and-protocols/      # ⬚ MCP servers/clients, function calling deep dive
│   ├── 14-agent-engineering/        # 🚧 agent loop in Py+TS (1/15) ★
│   ├── 15-autonomous-systems/       # ⬚ self-healing, autonomous loops, coding agents
│   ├── 16-multi-agent-and-swarms/   # 🚧 why multi-agent, comms protocols (2/14) ★
│   ├── 17-infrastructure/           # ⬚ serving, k8s, edge, CI/CD
│   ├── 18-ethics-safety-alignment/  # ⬚ red teaming, privacy, interpretability
│   └── 19-capstone-projects/        # ⬚ mini GPT, RAG system, agent swarm
├── outputs/                         # Registry for course-produced artifacts
│   ├── prompts/                     # (gitkeep — populated per-lesson)
│   ├── skills/                      #
│   ├── agents/                      #
│   └── mcp-servers/                 #
├── glossary/                        # terms.md, myths.md
├── site/                            # Static web app (app.js, build.js, data.js)
├── web/                             # (gitkeep)
├── assets/                          # Banner SVG
├── requirements.txt                 # Python dependencies
├── .github/workflows/               # deploy-site.yml (GitHub Pages)
├── ROADMAP.md                       # Per-lesson completion tracking
├── LESSON_TEMPLATE.md               # Template for new lessons
├── CONTRIBUTING.md                   # Contribution guide
└── FORKING.md                       # Fork guide for teams/schools
```

★ = Most architecturally interesting for AI/agent work

Each lesson follows: `NN-lesson-name/{code/, docs/en.md, outputs/, notebook/}`

## 5. Tech Stack

- **Primary languages**: Python (ML/DL), TypeScript (agents/tools), Rust (performance), Julia (math)
- **Python ML stack**: numpy, torch, torchvision, torchaudio, scikit-learn, pandas, matplotlib
- **LLM libraries**: transformers, tokenizers, accelerate, tiktoken
- **LLM API SDKs**: openai (>=1.0), anthropic (>=0.25)
- **Audio**: librosa, soundfile
- **Crypto**: Node.js built-in `crypto` module (Ed25519 for agent identity)
- **Package manager**: pip (requirements.txt), no lock file
- **Build system**: None (individual scripts), `node site/build.js` for the web app
- **CI/CD**: GitHub Actions (Pages deployment only)
- **Web**: Vanilla JS static site (no framework), GitHub Pages

## 6. Patterns Worth Borrowing

**Agent loop as the universal primitive** (`phases/14-agent-engineering/01-the-agent-loop/code/agent_loop.ts`): The tool registry pattern with `{ description, parameters, execute }` is clean and minimal. The TS version's type system (`ToolDef`, `Message`) is a good starting point for any agent implementation. Worth copying the structure directly.

**Protocol gateway composing A2A + ACP + ANP** (`phases/16-multi-agent-and-swarms/03-communication-protocols/code/main.ts`): The `ProtocolGateway` class shows how to layer identity verification → agent discovery → task delegation → audit logging into a single dispatch path. The `TaskManager` with `AsyncGenerator<TaskEvent>` for streaming task state is particularly well-designed.

**Skill files as agent-loadable knowledge** (`phases/*/outputs/skill-*.md`): The SKILL.md format — YAML frontmatter (name, description, version, tags) + markdown body with "when to use", "implementation checklist", "common mistakes" — is a practical format for injecting domain knowledge into AI agents at runtime.

**Prompt templates as lesson outputs** (`phases/*/outputs/prompt-*.md`): Each prompt template (e.g., `prompt-agent-debugger.md`) encodes domain expertise into a reusable system prompt. The agent debugger prompt's "common failure modes" list (infinite loops, wrong tool selection, context overflow) is directly useful.

**Fan-out orchestration** (`phases/16-multi-agent-and-swarms/01-why-multi-agent/code/single_vs_multi.ts`): The `Promise.all()` pattern for running independent specialist agents in parallel, then merging results for a downstream agent, is the simplest useful multi-agent pattern. The token-counting comparison against single-agent is a good way to justify the complexity.

**BPE tokenizer in three languages** (`phases/10-llms-from-scratch/01-tokenizers/code/`): Python for clarity, Rust for performance (with benchmarking), plus tiktoken comparison. Good reference implementation for understanding tokenization internals.

## 7. Gaps, Risks, and Limitations

**~77% of lessons are unwritten** — Only ~60 of 260+ lessons have code/docs. The most critical gaps for AI/agent work are: RAG (Phase 11), MCP servers/clients (Phase 13), context management/compression (Phase 14), autonomous loops (Phase 15), and most of the multi-agent patterns (Phase 16). The architecture is ambitious but the implementation is early-stage.

**No real LLM integration in any code** — All agent code stubs out the LLM call. There's no working example of connecting to OpenAI/Anthropic APIs despite both SDKs being in requirements.txt. The `agent_loop.py` even prints "To run with a real LLM, set ANTHROPIC_API_KEY and use agent_loop_real.py" — but that file doesn't exist.

**No tests** — Zero test files across the entire codebase. No pytest, no vitest, no test runner configuration. For a course that teaches ML pipelines and agent engineering, the absence of testing examples is notable.

**No package management for TypeScript** — TS files use Node.js built-in imports (`node:crypto`, `fs`, `child_process`) but there's no `package.json` or `tsconfig.json`. Running the TS code requires manual `npx tsx` or similar.

**No dependency pinning** — `requirements.txt` uses `>=` for everything. `torch>=2.0` could resolve to any version. No lock file means builds aren't reproducible.

**Flat outputs directory** — `outputs/index.json` is empty (`{"prompts": [], "skills": [], ...}`). The per-lesson outputs exist but aren't aggregated. The SkillKit integration mentioned in the README isn't wired up.

**No notebooks** — Despite `jupyter>=1.0` in requirements and `notebook/` directories in the lesson template, no `.ipynb` files exist in the repo.

**Single-contributor project** — All content appears to come from one author. The breadth of the curriculum (math → swarms in 4 languages) is extremely ambitious for a single contributor, which explains the completion gaps.

## 8. How to Run It

**Prerequisites:**
- Python 3.10+
- Node.js 20+ (for TypeScript lessons and site build)
- Rust toolchain (for Rust lessons in tokenizers)
- Julia (for math foundations Julia variants)

**Environment variables:**
- `ANTHROPIC_API_KEY` — referenced but not yet used in any working code
- `OPENAI_API_KEY` — referenced but not yet used

**Install + Run:**
```bash
git clone https://github.com/rohitg00/ai-engineering-from-scratch.git
cd ai-engineering-from-scratch

# Python setup
pip install -r requirements.txt

# Verify environment
python phases/00-setup-and-tooling/01-dev-environment/code/verify.py

# Run individual lessons (each is standalone)
python phases/01-math-foundations/01-linear-algebra-intuition/code/vectors.py
python phases/10-llms-from-scratch/01-tokenizers/code/bpe.py
python phases/07-transformers-deep-dive/02-self-attention-from-scratch/code/self_attention.py
python phases/14-agent-engineering/01-the-agent-loop/code/agent_loop.py

# TypeScript lessons (no package.json — use npx tsx directly)
npx tsx phases/14-agent-engineering/01-the-agent-loop/code/agent_loop.ts
npx tsx phases/16-multi-agent-and-swarms/01-why-multi-agent/code/single_vs_multi.ts
npx tsx phases/16-multi-agent-and-swarms/03-communication-protocols/code/main.ts

# Rust tokenizer
cd phases/10-llms-from-scratch/01-tokenizers/code && rustc bpe.rs -o bpe && ./bpe

# Build the web site
node site/build.js
```

**No build/test/lint commands exist** — each lesson is a standalone script. There is no unified build, test suite, or linter configuration.
