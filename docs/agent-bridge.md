# Agent bridge — pi ⇄ Quickshell

## 1. Transport

- **NDJSON** (newline-delimited JSON), one message per line, UTF-8.
- Unix socket at `$XDG_RUNTIME_DIR/bloub-island.sock` (falls back to
  `/tmp/bloub-island-<uid>.sock` if `XDG_RUNTIME_DIR` is unset).
- Writer: the pi extension (Node, `node:net`). Reader: `services/Agents.qml`
  (`Quickshell.Io.SocketServer` + `SplitParser`).
- Fire-and-forget: the extension never awaits a reply, never blocks pi. If the socket is
  missing, it retries with backoff and drops messages silently (never errors into the TUI).

Machine-readable schema: [`../protocol/v1.schema.json`](../protocol/v1.schema.json).

## 2. pi extension (`bloub-island.ts`)

Location: `~/.pi/agent/extensions/bloub-island.ts` (global) or `.pi/extensions/`
(project-local). Hot-reload with `/reload`. Quick test with `pi -e ./bloub-island.ts`.

Node builtins (`node:net`, `node:process`, …) are available in extensions
(see pi docs `docs/extensions.md` § Available Imports).

### Skeleton

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import net from "node:net";
import os from "node:os";

const SOCK = `${process.env.XDG_RUNTIME_DIR ?? os.tmpdir()}/bloub-island.sock`;
const ENABLED = process.env.BLOUB_ISLAND !== "0";
const V = 1;

let sock: net.Socket | null = null;
let backoff = 250;
let lastSent = 0;
let pending: Record<string, unknown> | null = null;

function connect() {
  if (!ENABLED || sock) return;
  const s = net.connect(SOCK);
  s.on("connect", () => { sock = s; backoff = 250; flush(); });
  s.on("error", () => { s.destroy(); sock = null; });
  s.on("close", () => { sock = null; setTimeout(connect, backoff); backoff = Math.min(backoff * 2, 5000); });
}

function flush() {
  if (pending && sock) { sock.write(JSON.stringify(pending) + "\n"); pending = null; }
}

const sessionId = () => process.env.PI_SESSION_ID ?? String(process.pid);

function send(event: string, extra: Record<string, unknown> = {}, opts: { throttle?: number } = {}) {
  const now = Date.now();
  if (opts.throttle && now - lastSent < opts.throttle) { pending = { v: V, event, sessionId: sessionId(), ts: now / 1000, ...extra }; return; }
  lastSent = now;
  const line = { v: V, event, sessionId: sessionId(), ts: now / 1000, pid: process.pid, ...extra };
  if (sock) sock.write(JSON.stringify(line) + "\n");
  else pending = line; // last-write-wins while disconnected
}

export default function (pi: ExtensionAPI) {
  connect();

  pi.on("session_start", async (_e, ctx) => {
    send("session_start", {
      cwd: process.cwd(),
      title: ctx.sessionManager?.getSessionId?.() ?? undefined,
      window: await findWindow(), // see § 6, optional
    });
  });
  pi.on("before_agent_start", async () => send("agent_start", { phase: "thinking" }));
  pi.on("turn_start", async (e) => send("turn_start", { phase: "streaming", turn: e.turnIndex }));
  pi.on("message_update", async () => send("message_update", { phase: "streaming" }, { throttle: 100 }));
  pi.on("message_end", async (e) => send("message_end", { role: e.message?.role }));
  pi.on("tool_execution_start", async (e) => send("tool_start", { phase: "tool", tool: e.toolName, toolCallId: e.toolCallId }));
  pi.on("tool_execution_end", async (e) => send("tool_end", { phase: e.isError ? "error" : "streaming", tool: e.toolName, toolCallId: e.toolCallId, ok: !e.isError }));
  pi.on("ui_prompt_start", async (e) => send("prompt_start", { phase: "waiting", kind: e.kind, title: e.title }));
  pi.on("ui_prompt_end", async () => send("prompt_end", { phase: "streaming" }));
  pi.on("agent_settled", async () => send("agent_settled", { phase: "done" }));
  pi.on("session_before_compact", async () => send("compact_start", { phase: "compacting" }));
  pi.on("session_compact", async () => send("compact_end", { phase: "streaming" }));
  pi.on("session_shutdown", async () => send("session_shutdown", { phase: "offline" }));
}
```

Notes:

- `ctx.isIdle()` is true in `agent_settled` unless another extension started a run — this is
  the event the pi docs recommend for status integrations (do **not** use `agent_end`, it
  fires before auto-retry/auto-compact/queued follow-ups).
- Throttle `message_update` (100 ms) — it fires per stream chunk.
- Catch everything: a broken socket must never surface in the TUI.

## 3. Event → phase catalogue

| pi hook | message `event` | `phase` | Extra fields |
|---|---|---|---|
| `session_start` | `session_start` | `idle` | `cwd`, `title`, `window?`, `terminal?` |
| `before_agent_start` | `agent_start` | `thinking` | |
| `turn_start` | `turn_start` | `streaming` | `turn` |
| `message_update` | `message_update` | `streaming` | (throttled) |
| `message_end` | `message_end` | – | `role` |
| `tool_execution_start` | `tool_start` | `tool` | `tool`, `toolCallId` |
| `tool_execution_update` | `tool_update` | `tool` | throttled |
| `tool_execution_end` | `tool_end` | `streaming` / `error` | `tool`, `ok` |
| `ui_prompt_start` | `prompt_start` | `waiting` | `kind`, `title` |
| `ui_prompt_end` | `prompt_end` | `streaming` | |
| `agent_settled` | `agent_settled` | `done` | |
| `session_before_compact` | `compact_start` | `compacting` | |
| `session_compact` | `compact_end` | `streaming` | |
| `session_shutdown` | `session_shutdown` | `offline` | |

Unknown events must be ignored by the QML side (forward compatibility).

## 4. Phase priority (QML side, single "hero" session)

Highest wins:

1. `waiting` — user attention required
2. `error`
3. `tool`
4. `thinking` / `streaming`
5. `compacting`
6. `done` (auto-expires after ~3 s)
7. `idle` / `sleep` (long inactive)

Session selection for the hero slot: highest phase priority, then most recent `ts`.
Others are collapsed into a `+N` badge (existing `badgeProviders` mechanism).

Staleness: if no message for **60 s** while phase ∈ {streaming, thinking, tool, compacting},
demote to `sleep`; after **10 min**, drop the session. `session_shutdown`/`offline` removes
it immediately. A dropped connection with no shutdown event still gets cleaned by the
staleness rule.

## 5. QML server side

`services/Agents.qml` (singleton) — sketch:

```qml
pragma Singleton
import Quickshell
import Quickshell.Io
import QtQuick

Singleton {
    id: root

    property var sessions: ({})      // sessionId -> { phase, tool, ts, cwd, ... }
    property string heroId: ""
    property bool busy: heroId !== "" && sessions[heroId]?.phase !== "done"

    readonly property var hero: sessions[heroId] ?? null

    function handleLine(line) {
        let msg; try { msg = JSON.parse(line) } catch (e) { return }
        if (msg.v !== 1 || !msg.sessionId) return
        // update registry, apply priority/staleness rules, prune
    }

    SocketServer {
        active: true
        path: `${Quickshell.env("XDG_RUNTIME_DIR")}/bloub-island.sock`
        handler: Component {
            Socket {
                parser: SplitParser {
                    splitMarker: "\n"
                    onRead: (line) => root.handleLine(line)
                }
            }
        }
    }
}
```

Verified API (Quickshell 0.2.1, `quickshell-io.qmltypes`):

- `SocketServer { active, path, handler }`, signal `onNewConnection`.
- `Socket` (prototype `DataStream`): `connected`, `path`, `write(data)`, `flush()`, parser.
- `SplitParser` (prototype `DataStreamParser`): `splitMarker`, signal `read(data: QString)`.
- `DataStream`: `parser` property.

Timers for staleness/`done` expiry must live in `Agents.qml` so they survive provider
switches in the island.

## 6. Click to focus the pi terminal (Phase 3)

The extension reports the terminal window address at `session_start`, because only it can
walk its own process ancestry:

1. Walk `/proc/<pid>/stat` parent chain from `process.pid` up to PID 1.
2. Run `hyprctl -j clients`, find the client whose `pid` is in that ancestor set.
3. Send `window: <address>` (and `terminal: <class>`) with the `session_start` message.

QML then calls `Hyprland.dispatch("focuswindow", "address:<addr>")`. Fallback when `window`
is missing: send `tty` and let the user configure a binding instead.

Privacy: send the address only, never window titles or workspace names.

## 7. Failure modes

| Failure | Behaviour |
|---|---|
| Quickshell starts after pi | Extension keeps retrying with backoff; on connect it re-sends `session_start` + current phase (last-write-wins pending slot) |
| pi killed with SIGKILL | No shutdown message; staleness rule cleans up (60 s demote, 10 min drop) |
| Stale socket file on disk | Qt removes/overwrites on `active: true` (QLocalSocket semantics); extension reconnect handles the swap |
| Partial line / malformed JSON | QML parser drops the line, keeps serving |
| Multiple pi sessions | Registry + priority rules; hero + `+N` badge |
| Socket flooded by streaming | Extension throttles; QML coalesces (only hero updates trigger re-render) |
| `BLOUB_ISLAND=0` | Extension is a complete no-op |

## 8. Security

- `XDG_RUNTIME_DIR` is `0700` — only the user can connect. No auth token needed; do not add
  one (the socket is single-user by construction).
- Messages carry metadata only: event names, tool names, cwd, ids. **Never** send prompt
  text, message contents, or secrets. If message snippets are ever wanted for the expanded
  view, make it opt-in and truncate aggressively.
- The extension only ever *writes*; it must not read from the socket.
