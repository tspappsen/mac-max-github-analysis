# Repository: zarazhangrui/follow-builders
**URL:** https://github.com/zarazhangrui/follow-builders

## 1. What It Does

A daily/weekly AI-curated digest of what top AI builders (researchers, founders, engineers) are posting on X/Twitter and saying on podcasts. Content is fetched centrally via GitHub Actions, pre-processed into JSON feeds, and then an LLM running on the user's own agent (OpenClaw or Claude Code) remixes it into a readable digest. Delivered via Telegram, email, or in-chat. No API keys required by end-users for the content itself — only for delivery.

Philosophy: "follow builders who build products and have original opinions, not influencers who regurgitate information."

## 2. AI/Agent Architecture

### The Pattern: Skill-as-System-Prompt

The repo is a **skill** — a structured set of instructions packaged for AI agents to consume. `SKILL.md` is the complete behavioral spec: it IS the system prompt. The agent (OpenClaw or Claude Code) reads it and knows exactly how to onboard users, schedule digests, customize prompts, and deliver content.

### Two-Phase Architecture

**Phase 1 — Deterministic (server-side, GitHub Actions):**
- `generate-feed.js` runs daily at 6am UTC via GitHub Actions
- Fetches tweets via X API (X_BEARER_TOKEN), YouTube transcripts via Supadata API
- Scrapes Anthropic/Claude blogs
- Deduplicates using `state-feed.json` committed back to the repo (7-day rolling window)
- Outputs `feed-x.json`, `feed-podcasts.json`, `feed-blogs.json` to the repo

**Phase 2 — LLM-based (client-side, user's agent):**
- `prepare-digest.js` runs on user's machine, fetches all three feeds + prompts from GitHub
- Assembles a single JSON blob with all content + prompts + config
- Agent's LLM reads the JSON and remixes it into a digest following the prompt instructions
- `deliver.js` sends it via Telegram, Resend email, or stdout

### No LLM in the pipeline — until the final remix step
The feed generation is entirely code-based. Zero LLM calls in `generate-feed.js`. The LLM is invoked exactly once: to take structured JSON and produce human-readable prose.

### Onboarding as Conversational Flow
The agent walks through a multi-step onboarding (Steps 1–9 in SKILL.md) entirely through natural language. No forms, no config files to edit manually. The agent writes `~/.follow-builders/config.json` for the user.

### Cron Scheduling via Agent
For OpenClaw: `openclaw cron add` with explicit channel/target detection. For Claude Code: system `crontab`. The skill explicitly handles the difference in platform capabilities.

## 3. Key Design Decisions

**Central feed, zero user dependencies:** All content fetching happens server-side (GitHub Actions). Users never need X_BEARER_TOKEN or SUPADATA_API_KEY. The only credentials users need are for *delivery* (Telegram bot, Resend). This drastically reduces setup friction.

**Prompt files as plain English, user-owned:** Five `.md` files in `prompts/` control all LLM behavior. Users can customize them conversationally ("make summaries shorter") — the agent copies the file to `~/.follow-builders/prompts/` and edits it. 3-tier priority: user-custom > GitHub remote > local default. Central prompt improvements auto-propagate unless the user has customized.

**Strict separation of data fetching and LLM remixing:** The `prepare-digest.js` script is intentionally dumb — just collects and assembles JSON. The SKILL.md explicitly says "Your ONLY job is to remix the content from the JSON. Do NOT fetch anything from the web." This avoids hallucination and confabulation of fake content.

**Anti-fabrication guardrails baked into prompts:** The digest-intro.md prompt says: "NEVER make up quotes, opinions, or content. If you don't have a link for something, do NOT include it. No link = not real = do not include." URL presence is used as a proxy for content validity.

**State in git, not a database:** Feed deduplication state (`state-feed.json`) is committed back to the repo by the GitHub Actions bot. No external DB needed. Pruned to 7 days automatically.

**Platform detection at runtime:** The skill auto-detects `openclaw` vs. `other` via `which openclaw`. Different cron setup, different delivery flow, different instructions — all branched in SKILL.md.

## 4. Project Structure

```
follow-builders/
├── SKILL.md                    # Agent behavioral spec (IS the system prompt)
├── README.md / README.zh-CN.md # User-facing docs (EN + Chinese)
├── config/
│   ├── default-sources.json    # Curated list of X handles + podcast URLs
│   └── config-schema.json      # JSON schema for config validation
├── prompts/                    # Plain-English LLM instructions
│   ├── summarize-tweets.md
│   ├── summarize-blogs.md
│   ├── summarize-podcast.md
│   ├── digest-intro.md
│   └── translate.md
├── scripts/
│   ├── generate-feed.js        # GitHub Actions feed generator (no LLM)
│   ├── prepare-digest.js       # Client-side: assembles JSON blob for LLM
│   ├── deliver.js              # Client-side: Telegram / Resend / stdout
│   └── package.json            # dotenv, proper-lockfile deps
├── feed-x.json                 # Generated tweet feed (committed by bot)
├── feed-podcasts.json          # Generated podcast feed (committed by bot)
├── feed-blogs.json             # Generated blog feed (committed by bot)
├── state-feed.json             # Dedup state (committed by bot)
├── examples/sample-digest.md   # Example output
└── .github/workflows/
    └── generate-feed.yml       # Daily 6am UTC cron job
```

## 5. Tech Stack

- **Runtime:** Node.js (ES modules, `import.meta.url`, native `fetch`)
- **Feed generation:** X API v2 (Bearer token), Supadata API (YouTube transcripts), HTML scraping for blogs
- **Delivery:** Telegram Bot API, Resend email API, stdout
- **Agent platforms:** OpenClaw, Claude Code (any agent that can read SKILL.md)
- **Scheduling:** GitHub Actions (feed gen), `openclaw cron` or system `crontab` (digest delivery)
- **Dependencies:** Only `dotenv` and `proper-lockfile` — intentionally minimal
- **Storage:** git-committed JSON files (no DB, no cloud storage)
- **Language support:** English, Chinese, bilingual (interleaved paragraph-by-paragraph)

## 6. Patterns Worth Borrowing for Max

**1. Skill-as-SKILL.md pattern:** Package agent behaviors as a single markdown file that IS the system prompt. No code needed for the agent logic — just well-structured natural language instructions with explicit branching (`if OpenClaw`, `if non-persistent agent`). This is a clean way to ship agent skills.

**2. Prepare-then-remix pattern:** Separate the deterministic data collection step (Node.js script → JSON blob) from the LLM remixing step. The LLM gets a single, clean JSON payload with everything it needs. No tool calls during generation. This is more reliable and cheaper than having the LLM make API calls itself.

**3. Central feed, zero client credentials:** Host the data fetching centrally (GitHub Actions) and expose it as public JSON. Users get the benefit without any API keys. Applicable to Max for any shared data source (e.g., a central knowledge feed).

**4. Prompt files as user-editable plain English:** Separate LLM instructions into `.md` files users can understand and customize conversationally. The agent mediates prompt editing rather than requiring users to touch code. The 3-tier priority (user > remote > local) is elegant.

**5. URL-as-validity-proxy:** Requiring every piece of content to have a URL link is a simple, enforceable anti-hallucination rule. If it doesn't have a URL, it didn't come from the feed, so don't include it.

**6. State in git:** For lightweight deduplication over time-series data, committing state back to the repo (via GitHub Actions bot) is a zero-infrastructure solution. Works well for daily/weekly cadences.

**7. Onboarding as multi-step conversation:** The 9-step onboarding flow in SKILL.md is a great pattern for any skill that needs configuration — ask questions one at a time, write config for the user, never require manual file editing.

## 7. Gaps, Risks, and Limitations

**Single point of failure — GitHub feeds:** Everything depends on the three JSON files in the repo being fresh. If the GitHub Actions job fails (API rate limits, token expiry), all users get stale content silently. No alerting mechanism.

**No LLM for curation quality:** The feeds contain raw tweets and transcripts. All signal/noise filtering relies entirely on the LLM's remixing prompts. There's no pre-filtering of low-quality content before it reaches the LLM.

**Centralized source curation:** The builder list is curated by the repo author and not user-modifiable. This is intentional (the "follow builders not influencers" philosophy) but limits personalization. Users can't add or remove sources.

**No persistence across sessions (Claude Code):** On non-persistent agents, there's no "memory" of previous digests. The `prepare-digest.js` state deduplication handles content-level dedup, but the agent has no conversational memory of user preferences beyond config.json.

**Supadata dependency:** YouTube transcript fetching relies on a third-party API (Supadata). Not open-source, has its own pricing/limits. If it changes or fails, podcast content disappears.

**No RAG:** Content is loaded in full into the LLM context. For large feeds with many builders and long transcripts, this could exceed context windows. No chunking or retrieval strategy.

**No error visibility to users:** `errors` field in the JSON blob is explicitly documented as "IGNORE these" in SKILL.md. Users won't know if parts of the feed failed to load.

**Markdown injection risk:** The digest delivery includes raw content from tweet text and podcast transcripts in the Telegram Markdown payload. If a tweet contains Telegram Markdown syntax, it could corrupt formatting (mitigated by the fallback to no parse_mode on failure).

## 8. How to Run It

### Prerequisites
- Node.js 20+
- An AI agent: OpenClaw (preferred) or Claude Code

### Installation

```bash
# For Claude Code
git clone https://github.com/zarazhangrui/follow-builders.git ~/.claude/skills/follow-builders
cd ~/.claude/skills/follow-builders/scripts && npm install
```

### Usage
1. Open Claude Code or OpenClaw
2. Say: "set up follow builders" or invoke `/follow-builders`
3. The agent guides you through: frequency → time → delivery method → language
4. First digest is generated immediately

### Manual digest
Type `/ai` at any time for an on-demand digest.

### Running feed generation locally (maintainer only)
Requires `X_BEARER_TOKEN` and `SUPADATA_API_KEY` in environment:
```bash
cd scripts
X_BEARER_TOKEN=xxx SUPADATA_API_KEY=yyy node generate-feed.js
```

### GitHub Actions
The workflow `.github/workflows/generate-feed.yml` runs daily at 6am UTC automatically. Secrets must be set in the repo's GitHub Actions secrets: `X_BEARER_TOKEN`, `SUPADATA_API_KEY`.
