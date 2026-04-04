# Repository: AnganSamadder/opentmux

**URL:** https://github.com/AnganSamadder/opentmux

## 1. What It Does

opentmux solves the "invisible agent" problem in multi-agent AI workflows: when OpenCode spawns sub-agents (child sessions), there is no native way to observe their execution in real-time. This plugin hooks into OpenCode's plugin event bus, listens for `session.created` events, and automatically opens a dedicated tmux pane per sub-agent that streams live output via `opencode attach <serverUrl> --session <id>`. Panes are auto-closed when sessions go idle, time out, or disappear. A smart CLI wrapper (`opentmux`) also auto-selects an available port, launches a new tmux session if you are not already in one, and can rotate or reclaim stale ports. The result is a persistent, multi-pane terminal workspace that mirrors exactly what every agent is doing—without any manual intervention.

## 2. AI/Agent Architecture

opentmux is not an agent itself; it is a **passive observer plugin** wired into OpenCode's plugin interface.

**Plugin contract (`src/types.ts`):**

```
Plugin = (ctx: PluginInput) => Promise<PluginOutput>
```

`PluginInput` provides:
- `ctx.client.session.status()` — polls the OpenCode server's `/session/status` REST endpoint
- `ctx.client.session.subscribe()` — event stream (available but not used directly; events arrive via the `event` handler)
- `ctx.serverUrl` — the HTTP URL of the running OpenCode server

**Event dispatch (`src/index.ts` → `TmuxSessionManager.onSessionCreated`):**
Every OpenCode event is forwarded to the plugin's `event` handler. The manager filters for `session.created` events where `properties.info.parentID` is set—meaning it only reacts to **child/sub-agent sessions**, not the root session.

**Pane lifecycle:**
1. `session.created` with `parentID` → enqueue a `SpawnRequest` (`src/spawn-queue.ts`)
2. `SpawnQueue` serializes spawns with configurable delay to avoid tmux race conditions
3. `spawnTmuxPane` (`src/utils/tmux.ts`) runs: `tmux split-window -h -d -P -F '#{pane_id}' 'opencode attach <url> --session <id>'`
4. `TmuxSessionManager` polls `/session/status` every 2 s; when a session is `idle`, missing, or timed out, it closes the pane via PID-level SIGTERM/SIGKILL + `tmux kill-pane`
5. `ZombieReaper` (`src/zombie-reaper.ts`) runs every 30 s, scanning all `opencode attach` processes system-wide and terminating any whose session ID no longer appears in `/session/status`—even across port restarts

**Context management:** none needed—the plugin carries no LLM context. All intelligence lives in OpenCode; opentmux only visualizes it.

## 3. Key Design Decisions

- **Plugin-not-agent pattern**: The whole system is a side-effect observer, not an actor. The `Plugin` type (`src/types.ts`) deliberately has no `tool` or `config` return—only `event`. This keeps the plugin strictly read-only with respect to the agent graph, so it can never accidentally influence agent behavior.

- **Serial spawn queue with debounced layout** (`src/spawn-queue.ts`, `src/tmux-session-manager.ts`): Tmux's `select-layout` is not idempotent under rapid pane creation. The queue serializes spawns with a configurable delay (`spawn_delay_ms`, default 300 ms), and layout recalculation is debounced (`layout_debounce_ms`, default 150 ms) and runs only after the queue fully drains. This pattern applies anywhere you drive a stateful external tool from async events.

- **Tmux layout string construction** (`src/layout.ts`, `src/utils/layout.ts`): Rather than relying on tmux's built-in `select-layout`, the plugin computes exact pixel-accurate tmux layout strings including the Adler-16 checksum tmux expects. This is necessary because built-in layouts do not support multi-column grids. Agents are distributed round-robin across columns (`groupAgentsByColumn`). The main pane percent shrinks as columns grow: 60% → 45% → 30%.

- **ZombieReaper with grace period** (`src/zombie-reaper.ts`): A session is only reaped after it fails `minZombieChecks` (default 3) consecutive scans. This prevents flapping from transient network hiccups. The reaper also implements **auto-self-destruct**: if no `opencode attach` client is seen for `selfDestructTimeoutMs` (default 1 hour), the server exits. Valuable for any long-running agent server that should not linger after all clients disconnect.

- **Port rotation and reclamation** (`src/bin/opentmux.ts`): When all ports in range 4096–4106 are occupied, the wrapper can optionally terminate the oldest process (`rotate_port: true`) or refuse to start. The reclaim logic inspects process TTY, tmux ancestry, and foreground status before sending signals—avoiding disruption of legitimate interactive sessions.

- **Zod for config validation** (`src/config.ts`): All config values are parsed through Zod schemas with defaults, so the plugin degrades gracefully when the config file is absent or partially malformed.

## 4. Project Structure

```
opentmux/
├── src/
│   ├── index.ts                   # Plugin entry point; wires config, manager, event handler
│   ├── types.ts                   # PluginInput/PluginOutput/Plugin type definitions
│   ├── config.ts                  # Zod schemas: PluginConfigSchema, TmuxConfigSchema, constants
│   ├── tmux-session-manager.ts    # ★ Core orchestrator: event intake, pane tracking, polling, cleanup
│   ├── spawn-queue.ts             # ★ Serialized async spawn queue with retry/delay
│   ├── zombie-reaper.ts           # ★ Background scanner; reaps orphaned opencode attach processes
│   ├── layout.ts                  # Pure functions: column distribution, layout string builder
│   ├── bin/
│   │   └── opentmux.ts            # CLI wrapper: port discovery, tmux session launcher, --reap flag
│   ├── scripts/
│   │   ├── install.ts             # Post-install: injects shell alias into .zshrc/.bashrc/fish
│   │   └── update-plugins.ts      # Background auto-updater spawned on each launch
│   └── utils/
│       ├── tmux.ts                # ★ Low-level tmux subprocess calls; spawnTmuxPane, closeTmuxPane
│       ├── layout.ts              # Tmux layout string math (checksums, cell trees, splitSizes)
│       ├── config-loader.ts       # Loads opentmux.json from project dir or ~/.config/opencode/
│       ├── process.ts             # OS helpers: getListeningPids, getProcessChildren, safeSignal
│       ├── logger.ts              # File-based logger (/tmp/opentmux.log)
│       └── index.ts               # Re-exports utils
├── src/__tests__/                 # Unit tests (Bun test runner)
├── docs/
│   └── LOCAL_DEVELOPMENT.md
├── AGENTS.md                      # AI agent contribution guidelines (coding standards for LLM agents)
├── PLAN.md                        # Design rationale and implementation notes
├── package.json                   # ESM module, tsup build, zod dependency
└── tsup.config.ts                 # Multi-entry build: index, bin, scripts
```

★ = AI-relevant modules

## 5. Tech Stack

- **Runtime**: Node.js (ESM), developed with Bun toolchain
- **Language**: TypeScript 5.x, strict mode
- **Build**: tsup (esbuild-based bundler, multi-entry)
- **Config validation**: Zod 3.x — runtime schema + type inference
- **IPC with OpenCode server**: Plain HTTP fetch to `/health`, `/session/status` REST endpoints
- **Terminal multiplexer**: tmux (spawned as subprocess via `node:child_process`)
- **LLM SDKs**: None — opentmux is infrastructure, not an LLM consumer
- **Vector stores / embeddings**: None
- **Framework**: OpenCode plugin API (minimal: event handler + optional tool/config)
- **Testing**: Bun test runner (unit tests in `src/__tests__/`)
- **CI/CD**: GitHub Actions — publish on `v*` tag push via `NPM_TOKEN`

## 6. Patterns Worth Borrowing

1. **Event-filter → spawn queue → lifecycle polling** (`tmux-session-manager.ts`): The three-phase pattern (filter event → enqueue work → poll for completion/cleanup) is directly applicable to any agentic system tracking sub-task panes, browser tabs, or containers. The `SpawnQueue` abstraction (`spawn-queue.ts`) is especially clean—it decouples the "when to spawn" decision from the "how to spawn" mechanism.

2. **Grace-period zombie detection** (`zombie-reaper.ts`, `markAsZombie`/`shouldKill`): The candidate-map pattern `Map<pid, {count, firstDetectedAt}>` where a process must fail N consecutive checks before being reaped is a robust pattern for any system that drives external processes based on polling. Prevents false positives from transient API failures.

3. **Checksum-bearing tmux layout strings** (`src/utils/layout.ts`, `buildMainVerticalMultiColumnLayoutString`): The full layout pipeline—`splitSizes` (Bresenham-style integer distribution), `LayoutCell` tree, `dumpLayoutCell`, `layoutChecksum`—is a self-contained pure-function implementation of the tmux layout wire format. Reusable for any tool that needs to programmatically control tmux geometry.

4. **Round-robin column distribution** (`src/layout.ts`, `groupAgentsByColumn`/`distributeAgentsRoundRobin`): A simple, well-tested pure function that maps N items to K columns evenly. Applicable to any UI that tiles N agent outputs.

5. **Smart binary wrapper with port discovery** (`src/bin/opentmux.ts`, `findAvailablePort`/`tryReclaimPort`): The pattern of probing ports, checking process ancestry, and reclaiming stale ones before launching a server is useful for any local AI dev server that might leave dangling processes.

6. **`AGENTS.md` as machine-readable contribution spec**: The project ships an `AGENTS.md` with explicit coding standards, import order, naming conventions, and async patterns targeted at AI coding agents, not humans. A lightweight way to make agentic code generation consistent without a linter.

## 7. Gaps, Risks, and Limitations

- **No automated integration tests**: `AGENTS.md` explicitly notes "no automated test suite." Unit tests cover config parsing and layout math but not the pane spawn/close lifecycle. A regression in `spawnTmuxPane` would only be caught manually.

- **Race: duplicate initialization guard is process-global** (`src/index.ts`, `isInitialized`): The `let isInitialized = false` guard lives in module scope. If OpenCode ever hot-reloads plugins or loads them in worker threads sharing the same module cache, this guard would incorrectly suppress initialization.

- **Polling instead of a true event subscription**: `TmuxSessionManager.pollSessions` polls `/session/status` every 2 s. OpenCode's `client.session.subscribe()` exists in the type definition but is not used for session lifecycle. Polling introduces up to a 2 s delay between session completion and pane close, and adds unnecessary HTTP traffic under high agent counts.

- **Layout string math is fragile at edge cases**: If tmux reports a window size of 0 (e.g., during session creation before first render), `buildMainVerticalMultiColumnLayoutString` silently produces an invalid string and layout application fails with no retry.

- **Auto-updater is fire-and-forget** (`src/scripts/update-plugins.ts`): The updater is spawned detached with `child.unref()`. If it fails or hangs it is invisible, and at scale could accumulate stale background processes.

- **ZombieReaper self-destruct exits the whole process**: If the reaper incorrectly concludes the server is abandoned (e.g., brief network blip), it calls `process.exit(0)`, terminating the OpenCode server itself. The `minZombieChecks` guard mitigates but does not eliminate this.

- **Windows support is partially tested**: Platform branches exist (`process.platform === 'win32'`, PowerShell paths) but CI only builds—no integration tests run on Windows.

- **No backpressure on spawn queue under burst**: `SpawnQueue` retries with exponential backoff but has a bounded max retry count. Under a burst of sub-agent creations with a slow tmux, panes can silently fail to spawn with no user-visible error beyond a log line.

## 8. How to Run It

**Prerequisites:**
- Node.js >= 18 (ESM required)
- `tmux` installed: `brew install tmux` / `sudo apt install tmux` / `winget install tmux`
- [OpenCode](https://opencode.ai) installed and on PATH

**Environment variables:**

| Variable | Purpose | Default |
|---|---|---|
| `OPENCODE_PORT` | Override server port | `4096` |
| `OPENCODE_TMUX_DISABLE_UPDATES` | Set to `1` to skip auto-update | unset |

No LLM API keys required — opentmux itself makes no LLM calls.

**Install:**
```bash
npm install -g opentmux
# Post-install script auto-injects shell alias into ~/.zshrc / ~/.bashrc / Fish config
```

**Configure OpenCode plugin** (`~/.config/opencode/opencode.json`):
```json
{ "plugin": ["opentmux"] }
```

**Optional config** (`~/.config/opencode/opentmux.json`):
```json
{
  "enabled": true,
  "port": 4096,
  "layout": "main-vertical",
  "main_pane_size": 60,
  "auto_close": true,
  "rotate_port": false,
  "max_ports": 10
}
```

**Run:**
```bash
opencode   # wrapper auto-selects port and launches tmux session if needed
```

**Clean up orphaned processes:**
```bash
opentmux --reap
```

**Local development:**
```bash
git clone https://github.com/AnganSamadder/opentmux
cd opentmux
bun install
bun run build      # outputs to dist/
bun run dev        # watch mode
bun run typecheck  # type-check only
```

**Logs:** `/tmp/opentmux.log`
