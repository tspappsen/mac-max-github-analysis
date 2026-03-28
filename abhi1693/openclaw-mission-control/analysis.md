# Repository: abhi1693/openclaw-mission-control

**URL:** https://github.com/abhi1693/openclaw-mission-control

## 1. What It Does

OpenClaw Mission Control is a **centralized control plane for operating fleets of LLM-powered agents**. It solves the hard operational problems that arise past a single agent:

- Multi-agent coordination across boards, groups, and gateways
- Human-in-the-loop  approval flows that gate agent actions before executiongovernance 
- Agent lifecycle  provision, wake, monitor, and deprovision agents with heartbeat liveness trackingmanagement 
 tags
- Multi-scope persistent  board-level and group-level tagged context storesmemory 
- Skills  install/update skill packs on agents at runtimemarketplace 
- Souls  browse and fetch agent persona templates from `souls.directory` (public registry)directory 
- Board  inbound HTTP endpoints routing external events directly to agentswebhooks 
- Cron  proactive agent behaviors triggered on schedule from the control planescheduling 
- Activity audit  immutable event stream for all board/task/agent actionsfeed 

Think of it as Kubernetes for AI  not the runtime itself, but the operator surface that manages fleets of agents running in external OpenClaw Gateway processes.agents 

## 2. AI/Agent Architecture

### Three-tier agent hierarchy

- **`agent-main`** (Gateway Main): One per gateway, no board assignment. Orchestrates across all boards, broadcasts to board leads, relays user messages via Slack/SMS, handles provisioning requests. Has exclusive `agent-main`-tagged API endpoints.
- **`agent-lead`** (Board Lead): One per board. Owns board-level  assigns tasks, enforces board rules, manages workers, creates/retires specialists, handles approvals. Has exclusive `agent-lead` API endpoints.coordination 
 review` tasks. Read board memory and group memory for context.

Role enforcement is structural, not convention: `OpenClawAuthorizationPolicy` checks `is_board_lead`, `board_id`, and `openclaw_session_id` bindings at the API layer. Workers literally cannot call lead endpoints.

### Gateway  how agents are actually wiredRPC 

Agents don't run inside Mission Control. They live in an external **OpenClaw Gateway** (WebSocket-based runtime). Mission Control communicates via `gateway_rpc. a typed WebSocket JSON-RPC client with ~60+ methods:py` 

```python
await openclaw_call("agents.files.set", {"agentId": key, "name": "SOUL.md", "content": ...})
await openclaw_call("chat.send", {"sessionKey": ..., "message": ..., "deliver": True})
await openclaw_call("agents.create", {...})
await openclaw_call("exec.approvals.resolve", {...})
await openclaw_call("skills.install", {...})
```

Mission Control is the operator controller; the gateway hosts the actual LLM sessions and agent runtime.

### Provisioning and the markdown-as-prompt pattern

`provisioning.py` uses Jinja2 to render agent workspace files and push them to the gateway via `agents.files.set`. The template map:

 review`), board rule injections, and explicit self-directed OpenAPI discovery instructions
- `IDENTITY.md. injects `agent_name`, `identity_role`, `identity_communication_style`, `identity_emoji`, `identity_purpose`, `identity_personality`, `identity_custom_instructions`j2` 
- `TOOLS.md. injects `BASE_URL`, `AUTH_TOKEN`, `AGENT_ID`, `BOARD_ID`, `WORKSPACE_ROOT`, and jq-based API discovery filtered by role tagj2` 
- `SOUL.md. persistent identity and values. Default text: *"You're not a chatbot. You're becoming someone."*j2` 
- `MEMORY.md. long-term memory file; agents read and write this themselves across sessionsj2` 
- `AGENTS.md. board agent roster for awareness of peer agentsj2` 
- `USER.md. user context for the boardj2` 

### Self-directed OpenAPI discovery as agent interface

Agents are explicitly instructed in `HEARTBEAT.md.j2` to fetch and parse the live OpenAPI spec at runtime:

```bash
curl -fsS "{{ base_url }}/openapi.json" -o /tmp/openapi.json
jq -r '... | select((.value.tags // []) | index("agent-lead")) | ...' /tmp/openapi.json | sort
```

The OpenAPI schemas carry LLM-semantic extension fields:
- `x-llm- machine-readable intent labelintent` 
- `x-when-to-use` / `x-when-not-to- agent decision guidanceuse` 
- `x-required- enforces role preconditions in the spec itselfactor` 
- `x-routing- routing semanticspolicy` 

This is a deliberate **schema-as-agent-interface** pattern. The API spec doubles as a living instruction  agents reason over it to decide which endpoints match their current objective.set 

### Memory architecture

- **Board  `board_memory` table; tagged text entries with `is_chat` flag; SSE streaming at 2s intervals; agents write back via `POST /api/v1/agent/boards/{board_id}/memory`Memory** 
- **Board Group  same structure scoped to a group; shared across related boardsMemory** 
- **Gateway workspace  `MEMORY.md` and `SOUL.md` in the agent's filesystem; durable, agent-owned long-term memory that persists across sessionsfiles** 
- **Agent  `identity_profile` (JSON), `identity_template`, `soul_template`, `heartbeat_config`, `last_seen_at`, `lifecycle_generation`state** 

Memory flows both ways: Mission Control pushes context at provision time; agents write back. This is filesystem-backed long-term memory with explicit agent-ownership semantics.

### Human-in-the-loop approvals

- `Approval` model: `action_type`, `payload` (JSON), `confidence` (float), `rubric_scores` (dict[str, int]), `status` (pending/resolved/rejected)
- Agents request approvals via `exec.approval.request` RPC; Mission Control holds them in a queue for human review
- On resolution, Mission Control calls `exec.approvals.resolve` back to the  a complete circuitgateway 
- Board rule `block_status_changes_with_pending_approval` structurally gates task state transitions

### Lifecycle state machine

 deleted`
- `lifecycle_generation`  detects stale lifecycle jobscounter 
- `wake_attempts`, `last_wake_sent_at`, `checkin_deadline_ liveness trackingat` 
- Background reconciliation via Redis RQ (`lifecycle_queue.py`, `lifecycle_reconcile.py`) for eventual consistency

## 3. Key Design Decisions

- **No LLM inference inside Mission  it's a pure control plane. All inference happens in the gateway. Mission Control controls behavior by updating markdown workspace files, not inference parameters.Control** 
- **Markdown files as the system prompt  the entire agent brain (identity, soul, tools, heartbeat protocol) is Jinja2-rendered markdown pushed to the gateway filesystem. Updating behavior = updating files.substrate** 
- **API tags as role-based access  the `agent-main`, `agent-lead`, `agent-worker` tag namespaces enforce role boundaries structurally in both the API layer and the spec itself. Agents can only discover and call endpoints appropriate to their role.control** 
- **Agent-authored  `SOUL.md` and `MEMORY.md` are explicitly designed for the agent to update across sessions. The agent is the author of its own long-term memory.memory** 
- **Coordination via structured natural-language  `coordination_service.py` builds messages with embedded JSON body templates and API URLs so agents can relay structured requests through the gateway main to humans over Slack/SMS, then parse the reply.messages** 
- **Ed25519 device  gateway auth uses persistent Ed25519 key pairs (stored server-side) to prove device continuity across reconnections.identity** 

## 4. Project Structure

```
openclaw-mission-control/
 backend/
 app/   
 api/           # FastAPI route handlers      
 agents.py, boards.py, tasks.py, approvals.py, gateways.py         
 board_memory.py, board_group_memory.py, board_webhooks.py         
 skills_marketplace.py, souls_directory.py, activity.py         
 core/          # Auth, config, rate limiting      
 agent_auth.py, agent_tokens.py, auth_mode.py         
 models/        # SQLModel table definitions      
 schemas/       # Pydantic schemas (with x-llm-* extension fields)      
 services/      
 openclaw/  # Core orchestration          
 gateway_rpc.py       # WebSocket JSON-RPC client              
 lifecycle_orchestrator.py              
 provisioning.py      # Jinja2 template rendering + push              
 coordination_service.py              
 device_identity.py   # Ed25519 key pairs              
 migrations/        # Alembic revisions   
 templates/         # Jinja2 agent workspace templates   
 BOARD_HEARTBEAT.md.j2        # Primary "system prompt"       
 BOARD_IDENTITY.md.j2       
 BOARD_SOUL.md.j2       
 BOARD_MEMORY.md.j2       
 BOARD_TOOLS.md.j2       
 frontend/
 src/   
 app/           # Next.js App Router pages       
 components/    # Atomic Design components       
 lib/           # Hooks, utils, agent-templates.ts       
 api/generated/ # Auto-generated TypeScript API client (orval)       
 compose.yml
 install.sh
```

## 5. Tech Stack

- **Backend**: Python 3.12+, FastAPI 0.131, SQLModel + SQLAlchemy (async), PostgreSQL 16, Alembic, Redis 7 + RQ, websockets, sse-starlette, Jinja2, Pydantic v2, Cryptography (Ed25519), uv package manager
 TS client codegen), Vitest + Testing Library, Cypress
- **Tooling**: Black + isort + ruff + flake8 + strict mypy (backend); ESLint + Prettier (frontend); Docker Compose
- **External**: OpenClaw Gateway (WebSocket runtime, not in this repo), `souls.directory` (public agent persona registry)

## 6. Patterns Worth Borrowing

- **Schema-as-agent-interface**: Annotate OpenAPI operations with `x-llm-intent`, `x-when-to-use`, `x-required-actor`. Agents fetch the live spec and filter by role tag to discover capabilities at  no hardcoded endpoint lists.runtime 
- **Markdown workspace files as system prompts**: Separate agent identity (JSON config) from rendered instructions (Jinja2 markdown). Update behavior by updating files, not code.
- **Multi-scope memory with ownership semantics**: Board memory (human-visible, tagged), group memory (shared across teams), and agent-owned `MEMORY.md` / `SOUL.md` files the agent updates itself. Three distinct layers, each with different access patterns.
 resolution propagated back to gateway session. Structurally gates state transitions.
- **Three-tier agent hierarchy with API-level role enforcement**: Main/Lead/Worker roles enforced via API tag namespaces, not just convention. The gateway discovers its allowed endpoints from the spec.
- **Board webhooks as external trigger channel**: Per-board inbound HTTP endpoints with HMAC verification routed to specific agents. Any external system can wake or task an agent.
- **Lifecycle reconciliation via background queue**: State machine + Redis RQ for eventual consistency across distributed agent lifecycle events.
- **`SOUL.md` as agent-authored persistent identity**: The agent explicitly owns and updates its identity file. Long-term memory with first-person authorship.

## 7. Gaps, Risks, and Limitations

- **No LLM observability inside Mission  no token counts, latency, cost tracking, or prompt/completion logging. All inference is a black box in the gateway.Control** 
- **No RAG or vector  memory is flat tagged text. No semantic search over board/group memory at scale.retrieval** 
- **No MCP (Model Context  tools are REST endpoints discovered via OpenAPI; no standardized tool/function-calling protocol.Protocol)** 
- **Gateway is a hard external  the entire value proposition requires a running OpenClaw Gateway. The gateway codebase is not in this repo and appears to be proprietary.dependency** 
- **Heartbeat protocol is  agents self-report via markdown task loops; no structured telemetry. Dead agents require timeout-based detection.brittle** 
- **Auth in local mode is a shared static  single `LOCAL_AUTH_TOKEN` for all users; no per-user identity in local mode.token** 
- **Souls directory is an external  `souls.directory` is a third-party dependency for persona templates; no fallback if unavailable.service** 
- **Limited multi- organizations exist but the data isolation model isn't deeply documented; Clerk mode adds user identity but RBAC is not granular beyond agent roles.tenancy** 

## 8. How to Run It

```bash
# One-liner (interactive installer)
curl -fsSL https://raw.githubusercontent.com/abhi1693/openclaw-mission-control/master/install.sh | bash

# Manual Docker
cp .env.example .env   # set LOCAL_AUTH_TOKEN (50+ chars) and BASE_URL
docker compose -f compose.yml --env-file .env up -d --build
# UI: http://localhost:3000 | API: http://localhost:8000/healthz

# Local dev
make setup
docker compose -f compose.yml --env-file .env up -d db
cd backend && uv run uvicorn app.main:app --reload --port 8000
cd frontend && npm run dev

# Auth modes
# AUTH_MODE= shared bearer token (single LOCAL_AUTH_TOKEN)local  
# AUTH_MODE= Clerk JWT with multi-user org supportclerk  
```
