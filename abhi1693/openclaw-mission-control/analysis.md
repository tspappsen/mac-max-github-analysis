# openclaw-mission-control — Analysis

**Repo:** [abhi1693/openclaw-mission-control](https://github.com/abhi1693/openclaw-mission-control)  
**Cloned:** `~/opensource/abhi1693/openclaw-mission-control`

---

## 1. What the Project Is and What It Does

OpenClaw Mission Control is the **centralized operations and governance platform for running OpenClaw** (an AI agent runtime system) across teams and organizations. It provides:

- **Work orchestration** — organizations → board groups → boards → tasks → tags hierarchy
- **Agent lifecycle management** — provision, wake, monitor, and deprovision agents with full state tracking (heartbeats, check-in deadlines, lifecycle generations)
- **Human-in-the-loop governance** — approval flows for sensitive agent actions before execution, with confidence scoring and rubric-based evaluations
- **Gateway management** — connect remote OpenClaw gateway nodes via WebSocket RPC; operate local and distributed execution environments from one UI
- **Activity auditing** — immutable activity event feed for all board/task/agent actions
- **Skills marketplace** — install/update skill packs on agents via gateway
- **Souls directory** — browse and fetch agent persona/identity templates from `souls.directory`
- **Board webhooks** — inbound HTTP webhooks routed to specific agents on a board
- **Onboarding wizard** — guided goal-setting conversational flow per board

This is the "control plane" for OpenClaw agents — not the agent runtime itself, but the operational surface for humans to manage fleets of agents.

---

## 2. Tech Stack and Key Dependencies

### Backend (Python/FastAPI)
- **FastAPI 0.131** — async REST API
- **SQLModel + SQLAlchemy (async) + PostgreSQL** — database ORM
- **Alembic** — database migrations
- **Redis + RQ (Redis Queue)** — background job queue for lifecycle reconciliation
- **websockets** — WebSocket client for gateway RPC protocol
- **sse-starlette** — Server-Sent Events for real-time streaming
- **Clerk (clerk-backend-api)** — JWT-based auth (optional mode; also has local shared-token mode)
- **Jinja2** — templating for agent identity/soul templates
- **Pydantic + pydantic-settings** — config and schema validation
- **Cryptography** — device identity key pairs for gateway auth
- **Python 3.12+**, strict mypy, Black, isort, ruff, flake8

### Frontend (Next.js)
- **Next.js 15** (App Router) — React framework
- **@clerk/nextjs** — Clerk auth integration
- **@tanstack/react-query** — data fetching and cache
- **@tanstack/react-table** — table management
- **@radix-ui/** — accessible UI primitives (dialog, popover, select, tabs, tooltip)
- **Tailwind CSS + tailwind-merge + class-variance-authority** — styling
- **recharts** — metric sparklines and charts
- **react-markdown + remark-gfm** — markdown rendering
- **cmdk** — command menu
- **lucide-react** — icons
- **Cypress** — E2E tests; **Vitest + Testing Library** — unit tests
- **orval** — OpenAPI-to-TypeScript client generation

### Infrastructure
- **Docker + Docker Compose** — full-stack containerized deployment
- **Interactive `install.sh`** — guided installer for `docker` or `local` mode

---

## 3. Project Structure Overview

```
openclaw-mission-control/
├── backend/
│   ├── app/
│   │   ├── api/           # FastAPI route handlers (one file per resource)
│   │   │   ├── agents.py, boards.py, tasks.py, approvals.py, gateways.py
│   │   │   ├── board_memory.py, board_group_memory.py, board_webhooks.py
│   │   │   ├── skills_marketplace.py, souls_directory.py, activity.py
│   │   │   └── gateway.py (thin WebSocket RPC proxy to gateway)
│   │   ├── core/          # Auth, config, rate limiting, security headers
│   │   │   ├── agent_auth.py, agent_tokens.py  # agent-specific auth
│   │   │   └── auth_mode.py  # local vs. clerk mode switching
│   │   ├── db/            # Session, CRUD base, pagination, query manager
│   │   ├── models/        # SQLModel table definitions
│   │   ├── schemas/       # Pydantic request/response schemas
│   │   └── services/
│   │       ├── openclaw/  # Core gateway/agent orchestration
│   │       │   ├── gateway_rpc.py       # WebSocket RPC client
│   │       │   ├── lifecycle_orchestrator.py
│   │       │   ├── lifecycle_queue.py / reconcile.py
│   │       │   ├── provisioning.py / provisioning_db.py
│   │       │   ├── coordination_service.py
│   │       │   ├── session_service.py
│   │       │   └── device_identity.py   # Ed25519 device key pairs
│   │       ├── webhooks/  # Inbound webhook dispatch
│   │       ├── activity_log.py
│   │       ├── souls_directory.py
│   │       └── queue.py / queue_worker.py  # RQ background jobs
│   ├── migrations/        # Alembic revisions
│   └── templates/         # Jinja2 agent identity/soul templates
├── frontend/
│   └── src/
│       ├── app/           # Next.js App Router pages
│       │   └── dashboard/{agents,boards,board-groups,approvals,
│       │                  gateways,activity,skills,tags,...}
│       ├── components/    # atoms/molecules/organisms (Atomic Design)
│       ├── lib/           # API client, utils, hooks
│       └── api/generated/ # Auto-generated TypeScript API client (orval)
├── docs/                  # Architecture, deployment, operations, policy docs
├── compose.yml            # Docker Compose full stack
├── install.sh             # Interactive installer
└── Makefile               # Dev workflow shortcuts
```

---

## 4. How to Run/Install

### One-liner bootstrap
```bash
curl -fsSL https://raw.githubusercontent.com/abhi1693/openclaw-mission-control/master/install.sh | bash
# Interactive: choose docker or local mode, auto-configures .env
```

### Docker Compose (manual)
```bash
cp .env.example .env          # set LOCAL_AUTH_TOKEN (50+ chars) and BASE_URL
docker compose -f compose.yml --env-file .env up -d --build
# UI: http://localhost:3000 | API: http://localhost:8000/healthz
```

### Local dev loop
```bash
make setup                    # install backend (uv) + frontend (npm) deps
docker compose -f compose.yml --env-file .env up -d db   # just postgres
cd backend && uv run uvicorn app.main:app --reload --port 8000
cd frontend && npm run dev
```

### Auth modes
- `AUTH_MODE=local` — shared bearer token (single `LOCAL_AUTH_TOKEN`)
- `AUTH_MODE=clerk` — Clerk JWT with multi-user organization support

---

## 5. Interesting Patterns and Features Relevant to Max

### Agent Identity and Soul Templates
- Each agent has an `identity_profile` (JSON), `identity_template` (Jinja2 text), and a `soul_template` — structured separation of persona config vs. rendered system prompts.
- Agents can be provisioned from **Souls Directory** (`souls.directory`) — a public registry of agent persona markdown files, browsable and installable from the UI. This is a live external catalog of agent identities/characters.
- `agent-templates.ts` in the frontend defines canned agent template types for quick creation.

### Hierarchical Memory (two levels)
- **BoardMemory** — persistent tagged text entries attached to a board, with an `is_chat` flag to distinguish chat history from knowledge notes. Has `source` field.
- **BoardGroupMemory** — same structure scoped to a board group (spans multiple boards). Enables shared memory across related agent teams.
- Memory items have `tags: list[str]` for lightweight categorization/retrieval.

### Approval-Gated Agent Actions (Human-in-the-Loop)
- `Approval` model stores: `action_type`, `payload` (JSON), `confidence` (float), `rubric_scores` (dict[str, int]), `status` (pending/resolved), `resolved_at`.
- Gateway-side: `exec.approval.request` / `exec.approval.resolve` RPC methods — agents can request approval at runtime before executing sensitive actions.
- Mission Control proxies approval decisions back to the gateway, forming a complete human-in-the-loop circuit.

### Cron Scheduling (Gateway-Side)
- Full cron management via gateway RPC: `cron.list`, `cron.add`, `cron.update`, `cron.remove`, `cron.run`, `cron.runs`, `cron.status`.
- `cron` is also a gateway event — the runtime can push cron-tick events back to Mission Control.
- This enables **proactive, scheduled agent behaviors** managed from the control plane.

### Board Webhooks (External Trigger Channel)
- `BoardWebhook` model: per-board inbound HTTP endpoints, each optionally routed to a specific `agent_id`, with HMAC signature verification (`secret` + `signature_header`).
- Enables **external systems to trigger agents** — e.g., a GitHub webhook, a monitoring alert, a pipeline event routes directly into an agent's board.

### Gateway WebSocket RPC Protocol
- Mission Control connects to each gateway via WebSocket with a versioned protocol (v3).
- Supports two connection modes: `device` (Ed25519 signed device identity + pairing) and `control_ui` (simple token auth).
- Device identity: persistent Ed25519 key pair stored on the server; signs auth payloads to prove device continuity across sessions.
- Full `chat.send` / `chat.history` / `chat.abort` methods — Mission Control can **directly inject messages into agent chat sessions**.
- `sessions.compact` — trim/summarize session history from the control plane.
- `talk.mode` method + event — voice/talk mode control.

### TTS (Text-to-Speech) Control
- Gateway exposes `tts.status`, `tts.providers`, `tts.enable/disable`, `tts.setProvider`, `tts.convert` — voice output management from the control plane.

### Voice Wake Word
- `voicewake.get/set` methods and `voicewake.changed` event — manage wake word configuration for agents.

### Multi-Node Orchestration
- `node.list`, `node.describe`, `node.invoke`, `node.pair.*` — gateways can orchestrate sub-nodes; Mission Control can reach through to invoke tasks on specific nodes.
- `node.invoke.request` / `node.invoke.result` events enable async task dispatch and result collection across a node graph.

### Agent Lifecycle State Machine
- Agent status: `provisioning → active → sleeping → deleted` (inferred from fields).
- `lifecycle_generation` counter — detects stale lifecycle jobs.
- `wake_attempts`, `last_wake_sent_at`, `checkin_deadline_at` — active liveness tracking with deadline enforcement.
- Background reconciliation via Redis RQ (`lifecycle_queue.py`, `lifecycle_reconcile.py`) — eventual consistency for agent state.

### Activity Feed
- `ActivityEvent` model tied to board/task/agent context — queryable timeline for debugging and accountability.
- Frontend has a `/activity` route with real-time feed.

### Skills Marketplace
- `skills.status`, `skills.bins`, `skills.install`, `skills.update` — skill pack management via gateway from Mission Control.
- Frontend `MarketplaceSkillsTable` and `SkillPacksTable` expose browsing and install UI.

### Mentions Service
- `app/services/mentions.py` — parses `@mention` references in task content, enabling agent-to-agent or human-to-agent tagging within task descriptions.

### Board Snapshot / Group Snapshot
- `board_snapshot.py`, `board_group_snapshot.py` — capture complete board or group state as a serializable snapshot. Useful for context injection (feeding full board context to an agent) or backup.

### Custom Task Fields
- Tasks support arbitrary custom fields — extensible metadata per task, managed via `task_custom_fields.py`.

### Lead Agent Policy
- `lead_policy.py`, `is_board_lead` flag on Agent — designates a "lead" agent per board with special policy authority (likely first-to-receive tasks or authority to delegate).

### Onboarding Wizard
- `BoardOnboardingSession` stores a conversational onboarding flow with `messages` (JSON array) and `draft_goal` — a structured goal-setting session before the board goes live.

---

## Summary for Max

The most reusable ideas from this codebase:

| Pattern | What It Is | Value for Max |
|---|---|---|
| **Board/Group Memory** | Persistent tagged text memory at board + board-group scope | Multi-scope memory hierarchy; `is_chat` flag to separate episodic from semantic memory |
| **Souls Directory** | Public registry of agent personas (markdown, via `souls.directory`) | Ready-made catalog for loading agent identities; could seed Max's persona system |
| **Approval flow** | Confidence + rubric scores → pending approval → resolved decision | Pattern for Max to pause and seek human confirmation on high-stakes actions |
| **Cron scheduling** | Gateway-managed cron jobs with event push-back | Proactive/scheduled agent actions (reminders, reports, monitoring) |
| **Board Webhooks** | Inbound HTTP triggers routed to a specific agent | Any external system can wake Max; integrates with GitHub, monitoring, etc. |
| **chat.send injection** | Direct message injection into agent sessions from control plane | Control plane can push context, corrections, or triggers into active sessions |
| **Node invoke** | Async task dispatch across a node graph with result collection | Multi-agent delegation pattern; Max as orchestrator or sub-node |
| **Lifecycle state machine** | Provision/wake/sleep/delete with heartbeat + deadline enforcement | Robust agent health management pattern |
| **Board Snapshot** | Full board context serialization | Feed complete project context to agent as a structured prompt |
| **@mentions in tasks** | Agent/human mentions parsed in task content | Lightweight agent-to-agent communication via task descriptions |
| **Identity template + soul template** | Jinja2-rendered system prompts from profile JSON | Separates persona data from rendered instructions; enables dynamic identity |
