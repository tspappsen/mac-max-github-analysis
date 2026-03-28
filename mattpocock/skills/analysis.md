# Repository: mattpocock/skills

**URL:** https://github.com/mattpocock/skills

---

## 1. What It Does

`mattpocock/skills` is a curated library of **AI agent skill definitions** — reusable, structured prompt packages that extend the capabilities of AI coding agents (primarily Claude Code). Each skill is a directory containing a `SKILL.md` file (and optionally supporting docs and scripts) that the agent loads to execute a specific workflow.

Skills are installed into a project via an external CLI:

```
npx skills@latest add mattpocock/skills/<skill-name>
```

Once installed, the agent reads the skill's frontmatter description to decide when to auto-activate it, and follows the SKILL.md instructions to carry out the task. Skills span four domains:

- **Planning & Design** — PRDs, refactor plans, interface design, issue breakdowns
- **Development** — TDD, bug triage, architecture improvement, test migration
- **Tooling & Setup** — pre-commit hooks, git safety guardrails
- **Writing & Knowledge** — article editing, domain glossaries, Obsidian notes

---

## 2. AI/Agent Architecture

### Skill-as-Prompt Pattern

Each skill is fundamentally a **prompt fragment with metadata**. The YAML frontmatter block at the top of every `SKILL.md` contains:

```yaml
---
name: skill-name
description: What it does. Use when [trigger conditions].
---
```

The `description` field is load-bearing: it's what the agent reads (via system prompt injection or tool registry) to decide which skill to activate. Matt explicitly calls this out in `write-a-skill`:

> "The description is **the only thing your agent sees** when deciding which skill to load."

### Agent Selection = Semantic Dispatch

Skills are dispatched semantically. The description acts as a **routing predicate** — the agent matches user intent to the correct skill by comparing the description against the current request. Well-crafted descriptions include explicit trigger keywords (e.g., `"mentions 'grill me'"`, `"mentions tracer bullets"`).

### Sub-agent Parallelism

Several skills explicitly spawn parallel sub-agents:

- **`design-an-interface`**: runs multiple sub-agents concurrently to generate radically different interface designs, then converges by comparing them — a fan-out/fan-in agentic pattern.
- **`triage-issue`**: uses `subagent_type=Explore` to deep-investigate the codebase before surfacing findings.

### Tool Integration

Skills integrate tightly with external tools:
- **GitHub CLI (`gh`)** — `write-a-prd`, `prd-to-issues`, `triage-issue`, `qa` all file GitHub issues directly.
- **Pre-commit hooks** — `setup-pre-commit` wires Husky + lint-staged + Prettier + type-check + tests.
- **Claude Code hooks** — `git-guardrails-claude-code` injects a `PreToolUse` hook via a bash script that intercepts Claude's tool calls and blocks dangerous git commands (`push`, `reset --hard`, `clean -f`, `branch -D`, etc.) before they execute.

### Progressive Disclosure via Multi-File Skills

Long-form content is split into separate files to control context window usage:

```
tdd/
├── SKILL.md          ← main workflow (lean)
├── tests.md          ← good/bad test examples
├── mocking.md        ← when/how to mock
├── deep-modules.md   ← architectural philosophy
├── interface-design.md
└── refactoring.md
```

The agent loads `SKILL.md` first; detailed reference files are linked and loaded only when needed.

---

## 3. Key Design Decisions

### Decisions Unique to This Repo

1. **The `description` field as the sole routing mechanism.** No configuration files, no explicit skill mappings — the agent does pure semantic dispatch. This makes skills portable across agents that support the skills format.

2. **Vertical slices / tracer bullets as the universal planning primitive.** Almost every planning skill (`prd-to-plan`, `prd-to-issues`, `tdd`) mandates tracer-bullet vertical slices. This is a deliberate architectural stance against horizontal (batch) development.

3. **Deep modules as the architectural north star.** The TDD skill, `improve-codebase-architecture`, and `write-a-prd` all reference "A Philosophy of Software Design" (Ousterhout) directly. Deep modules (small interface, deep implementation) are the preferred design target.

4. **Skills under 100 lines by default.** The `write-a-skill` meta-skill prescribes that `SKILL.md` should stay under 100 lines, using linked files for overflow. This is a token-budget design pattern.

5. **Deterministic operations → scripts, not generated code.** Where an operation is predictable and repeatable (e.g., blocking dangerous git commands), it's encoded as a script, not left to the model to re-generate. This improves reliability and saves tokens.

6. **No runtime dependencies in the repo itself.** No `package.json`, no `tsconfig`, no build step. The repo is pure content; the runtime is provided by the external `skills` CLI.

---

## 4. Project Structure

```
mattpocock/skills/
├── README.md                         ← Catalog of all skills with install commands
├── LICENSE                           ← MIT (2026, Matt Pocock)
│
├── design-an-interface/SKILL.md      ← Design It Twice via parallel sub-agents
├── edit-article/SKILL.md             ← Article restructure + prose tightening
├── git-guardrails-claude-code/
│   ├── SKILL.md                      ← Setup instructions
│   └── scripts/block-dangerous-git.sh ← PreToolUse hook script
├── grill-me/SKILL.md                 ← Relentless Socratic design interview
├── improve-codebase-architecture/
│   ├── SKILL.md                      ← Workflow for finding shallow modules
│   └── REFERENCE.md                  ← Dependency categories + issue template
├── migrate-to-shoehorn/SKILL.md      ← Replace `as` assertions with shoehorn
├── obsidian-vault/SKILL.md           ← Obsidian note CRUD (hardcoded vault path)
├── prd-to-issues/SKILL.md            ← PRD → GitHub issues
├── prd-to-plan/SKILL.md              ← PRD → phased plan Markdown file
├── qa/SKILL.md                       ← Conversational QA → GitHub issues
├── request-refactor-plan/SKILL.md    ← Refactor RFC → GitHub issue
├── scaffold-exercises/SKILL.md       ← Course exercise directory scaffolding
├── setup-pre-commit/SKILL.md         ← Husky + lint-staged + Prettier setup
├── tdd/
│   ├── SKILL.md                      ← Red-green-refactor workflow
│   ├── deep-modules.md
│   ├── interface-design.md
│   ├── mocking.md
│   ├── refactoring.md
│   └── tests.md
├── triage-issue/SKILL.md             ← Bug → root cause → GitHub issue
├── ubiquitous-language/SKILL.md      ← DDD glossary extraction → UBIQUITOUS_LANGUAGE.md
├── write-a-prd/SKILL.md              ← Interview → PRD → GitHub issue
└── write-a-skill/SKILL.md            ← Meta-skill for creating new skills
```

**15 skills total.** No nesting beyond one level. No shared library code.

---

## 5. Tech Stack

| Layer | Technology |
|---|---|
| Skill format | Markdown + YAML frontmatter |
| Delivery | `npx skills@latest` (external CLI, not in this repo) |
| Target agent | Claude Code (primary), likely compatible with any MCP/skills-format agent |
| Tool integration | GitHub CLI (`gh`), Husky, lint-staged, Prettier, Bash |
| Hook mechanism | Claude Code `PreToolUse` hooks (JSON stdin, exit code semantics) |
| TypeScript tooling referenced | `@total-typescript/shoehorn`, `pnpm`, `pnpm ai-hero-cli` |
| No runtime | No Node.js, no package.json, no build pipeline |

---

## 6. Patterns Worth Borrowing

### 1. Description-as-Dispatcher
The `name` + `description` frontmatter pattern is immediately reusable. Write skill descriptions as: _"[capability]. Use when [trigger keywords/contexts]."_ The trigger clause is what makes semantic dispatch reliable.

### 2. Token Budget via Progressive Disclosure
Keep the main skill file lean (< 100 lines). Offload deep reference material to linked sibling files. The agent loads only what it needs, when it needs it — a clean working memory management pattern.

### 3. Tracer Bullet as Mandatory Planning Primitive
Requiring vertical slices in every planning skill prevents the common failure mode of writing all tests or all boilerplate before any real behavior exists. This is an opinionated default worth encoding into any planning workflow.

### 4. PreToolUse Safety Hooks
The `block-dangerous-git.sh` approach — reading JSON from stdin, pattern-matching the command, exiting 2 to block — is a clean, portable safety primitive for any agentic coding environment. The pattern generalizes to any class of dangerous tool calls.

### 5. Meta-Skill for Skill Creation
`write-a-skill` is a skill for writing skills. This self-referential loop ensures new skills conform to the same structural constraints (description format, line limit, split-file rules) without needing separate documentation.

### 6. DDD Ubiquitous Language as a Skill
Externalizing the domain glossary extraction as a skill (`ubiquitous-language`) makes terminology hardening a first-class workflow step rather than an afterthought. The UBIQUITOUS_LANGUAGE.md output artifact creates a durable shared reference.

### 7. Sub-Agent Fan-Out for Divergent Thinking
`design-an-interface` uses parallel sub-agents to force multiple distinct designs before converging. This is a low-friction way to implement "Design It Twice" in agentic workflows.

---

## 7. Gaps, Risks, and Limitations

### Hardcoded Personal Context
`obsidian-vault/SKILL.md` contains a hardcoded vault path: `/mnt/d/Obsidian Vault/AI Research/`. This is clearly Matt's personal WSL2 environment. Anyone installing this skill would need to override the path manually. This is the one skill that isn't cleanly portable.

### No Versioning or Composability Primitives
Skills have no declared version, no dependency declarations, and no composability metadata. If `prd-to-issues` evolves to require `write-a-prd` to have run first, there's no machine-readable way to express that constraint.

### External CLI is a Black Box
The `npx skills@latest` delivery mechanism is not in this repository. There's no documentation for how skills are actually injected into the agent's context (system prompt? tool registry? file drop?). The install mechanism is fully opaque from inspection of this repo alone.

### No Testing for Skills Themselves
There's no mechanism to validate that a skill works as intended. Prompt quality is untested and relies entirely on human authorship discipline.

### Scaffold-Exercises is Project-Specific
`scaffold-exercises` references `pnpm ai-hero-cli internal lint` — a custom CLI from Matt's AI Hero course project. This skill is only useful within that specific project context and is essentially a personal workflow script exposed as a public skill.

### Token Costs Not Bounded for Long Workflows
Skills like `write-a-prd` and `request-refactor-plan` involve multiple interview rounds, codebase exploration, and issue filing. There are no guardrails on context window usage or round-trip count. In a large codebase, these could be expensive.

### `improve-codebase-architecture` Dependency Classification is Manual
The `REFERENCE.md` dependency taxonomy (in-process, local-substitutable, remote-but-owned, true external) is well-designed but entirely relies on the agent classifying correctly. No scaffolding or heuristics are provided to guide classification in ambiguous cases.

---

## 8. How to Run It

### Install a Skill into Your Project

```bash
npx skills@latest add mattpocock/skills/<skill-name>
# Examples:
npx skills@latest add mattpocock/skills/tdd
npx skills@latest add mattpocock/skills/write-a-prd
npx skills@latest add mattpocock/skills/git-guardrails-claude-code
```

The `skills@latest` CLI (separate package, not in this repo) handles fetching and injecting the skill into your agent's context.

### Use a Skill
After installation, invoke naturally in your agent chat:
- "Let's do TDD on this feature" → activates `tdd`
- "Write a PRD for X" → activates `write-a-prd`
- "Grill me on this design" → activates `grill-me`
- "Triage this bug" → activates `triage-issue`

### Set Up Git Guardrails Manually (Claude Code)
If you want to wire the git safety hook directly without the CLI:

1. Copy `git-guardrails-claude-code/scripts/block-dangerous-git.sh` into your project
2. Add a `PreToolUse` hook in your Claude Code config pointing to this script
3. The script reads tool input JSON from stdin and exits `2` (block) if dangerous git patterns are detected

### Browse Skills Without Installing
All skills are plain Markdown — just read the SKILL.md files directly for the prompt content and methodology.
