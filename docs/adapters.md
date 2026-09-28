# Adapters — supporting more than pi

Short answer to *"do we need to build a native agent app first?"*: **no.** What the island needs
is a stable status contract plus one adapter per harness. A native agent app is a different
product (runtime, sandboxing, model credentials, governance) and months of work; the island is
a *view* and must stay dumb.

## 1. Three layers, one rule

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Harness — produces events                                │
│    pi · Claude Code · Codex · opencode · OpenBot ·          │
│    OpenMausBot · anything speaking AG-UI                    │
└───────────────┬─────────────────────────────────────────────┘
                │ native hooks / SSE / NDJSON logs / plugins
┌───────────────▼─────────────────────────────────────────────┐
│ 2. Adapter — normalizes into protocol v1                    │
│    bloub-island.ts (pi) · openmaus SSE · AG-UI client · …   │
└───────────────┬─────────────────────────────────────────────┘
                │ NDJSON over $XDG_RUNTIME_DIR/bloub-island.sock
┌───────────────▼─────────────────────────────────────────────┐
│ 3. Island — pure view                                       │
│    Agents.qml → DiAgent.qml → BloubAvatar.qml               │
└─────────────────────────────────────────────────────────────┘
```

**Rule:** layer 3 never learns about a harness. No harness names, no HTTP clients, no log
parsing in QML beyond protocol v1. Every new capability is an adapter.

## 2. Prior art worth copying

Both projects below are *runtimes* (they own agent processes). Neither is needed for the
island, but both validate the adapter pattern — and their event streams can be consumed
as-is by an adapter.

### OpenMausBot — <https://github.com/milind-soni/OpenMausBot>

- 3.6 k★, **Apache-2.0**, Electron chat app (macOS/Windows/Ubuntu), Claude/Codex/Grok CLIs
  under the hood. Local-first.
- Architecture (from its README): *"The app holds no transports of its own — it sends typed
  commands over HTTP and folds one SSE event stream into state. The harness server owns every
  agent process and normalizes each provider's native protocol into one canonical runtime
  event stream (logged per-thread as NDJSON)."*
  - Harness server: `127.0.0.1:8799`, HTTP + **one SSE stream**.
  - Drivers per provider (`server/drivers/`): Claude, Codex, Grok Build via stream-JSON /
    JSON-RPC / ACP.
  - Event logs: NDJSON per thread under `~/.openmausbot`.
  - Also ships a stdio MCP server for external clients.
- **This is the adapter architecture already built.** Our island can be one more client: an
  adapter that subscribes to the SSE stream (or tails the NDJSON logs) and re-emits protocol
  v1. No fork, no native app, no Electron needed on our side.
- Caveat for Linux users: host control on Ubuntu **Wayland is disabled** (their issue #345) —
  irrelevant when we only consume status events.

### OpenBot (CopilotKit) — <https://github.com/CopilotKit/openbot>

- 5.6 k★, **MIT**, TypeScript. Self-hosted "AI coworkers" platform: Docker Compose +
  PostgreSQL + Bun + CopilotKit Intelligence. Alpha, "a template, not a product".
- Every Bot is **any endpoint speaking AG-UI** (<https://github.com/ag-ui-protocol/ag-ui>),
  the open agent-to-user interaction protocol: ~16 standard event types (run / step /
  text-message / tool-call lifecycle, state snapshots/deltas), transport-agnostic
  (SSE, WebSocket, webhooks).
- Governance: every tool call passes a gateway that decides + records before acting.
- Adapter shape: subscribe to the AG-UI run stream, map run/tool events to phases, write
  protocol v1. Feasible without forking; the stack is heavy to *run* (Docker + Postgres), so
  treat it as "support if present", not a dependency.

## 3. Adapter feasibility table

| Harness | Mechanism | Effort | Notes |
|---|---|---|---|
| **pi** | extension (`~/.pi/agent/extensions/`, this repo's reference adapter) | done in Phase 1 | richest event set (14 hooks) |
| **OpenMausBot** | SSE `127.0.0.1:8799` **or** tail `~/.openmausbot/**/*.ndjson` | small | already canonical; MIT-quality docs; good second adapter |
| **OpenBot** | AG-UI event stream over SSE | small–medium | map AG-UI lifecycle → phases; Docker stack must be running |
| **Claude Code** | hooks (settings) → small script → socket | small | verify hook payload shape first |
| **Codex CLI** | `notify` hook → script → socket | small | verify |
| **opencode** | plugin → socket | small | verify |
| **anything else** | wrapper script + `socat`/`nc` | tiny | last resort: `echo '{"v":1,…}' \| socat - UNIX-CONNECT:$SOCK` |

## 4. Where adapters run (pick per complexity)

1. **Direct writers (v1 default).** Every adapter connects to the socket itself (as the pi
   extension does). Zero infrastructure, zero extra processes. Works for hook-based harnesses.
2. **Local router (recommended once adapter #2 needs polling/tailing).** A tiny
   `systemd --user` service that fans in sources needing a long-lived client (SSE, log
   tailing) and writes one socket. It is *not* an agent runtime — it owns no agents, no
   credentials, no sandbox. Think "status multiplexer", ~150 lines.
3. **Inside Quickshell (only for trivial cases).** `Agents.qml` may watch one file with
   `FileView { watchChanges: true }` or run one `curl -N` process via `Quickshell.Io.Process`.
   Convenient, but do not let the shell become the integration hub — that is what layer 2 is
   for.

Decision: ship **option 1** in v1; adopt **option 2** when the second adapter lands; allow
option 3 only for a single degenerate case (a plain NDJSON file, no network).

## 5. Protocol changes for multi-harness (already applied, backwards compatible)

Two optional fields were added to protocol v1:

```json
{"v":1,"agent":"pi","source":"extension","event":"agent_start","sessionId":"…","ts":…}
```

- `agent` — harness id: `pi`, `openmausbot`, `openbot`, `claude-code`, `codex`, `opencode`, …
  Defaults to `pi` when absent (existing writers keep working unchanged).
- `source` — how it got here: `extension` | `sse` | `log` | `hook` | `script`.

UI rules once more than one harness is present: the hero slot is chosen by phase priority
first, then recency **within the same project/cwd**; sessions from different harnesses are
distinguished by a tiny badge (the `agent` field → icon in `iconForProviderId`).

## 6. Why not build the native app

Building an own agent runtime (à la OpenBot/OpenMausBot) means owning: process supervision,
per-agent sandboxes/containers, model credentials and key storage, browser automation,
tool governance + audit, update channel, packaging for 3 OSes. None of that is needed to show
a mascot in a bar. If the goal ever becomes "own AI coworker product", do it as a separate
project and make the island its view — the protocol in this repo is already the seam that
makes that swap cheap.
