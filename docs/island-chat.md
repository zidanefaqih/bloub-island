# Island chat — Grok Bot as the island's face

## 0. Intent

The user's goal, verbatim intent: **Grok Bot lives only in the dynamic island.** The backend
app runs in the tray; the island is where the mascot lives, where status shows, and where you
can *chat*. Opening the full app window is optional, never required.

Current island behavior to build on (end4-pC):

- Center pill shows the clock/date (`DiIdle`).
- Hover expands the island into an overlay (calendar, uptime, tasks) — screenshot-verified.
- Other providers (media, OSD, notifications) already swap into the pill.

## 1. Backend decision: OpenMausBot, not OpenBot

| | OpenMausBot | OpenBot (CopilotKit) |
|---|---|---|
| Shape | **Native desktop app (Electron)**, macOS/Windows/Ubuntu builds | **Web app** served by a platform (Docker + Postgres + Bun + Intelligence) |
| Tray | Yes — `electron/system-tray.mjs`, `docs/verification/startup-tray.md` | No tray; it is a hosted web UI |
| API for the island | **HTTP commands + one SSE event stream** on the harness server (`127.0.0.1:8799`) | AG-UI events, but you must operate the platform |
| License | Apache-2.0 | MIT (app), but the stack is heavier |
| Fits "app in tray, island chats" | **Yes** | No — it is browser-first by design |

**Verdict: OpenMausBot.** OpenBot is disqualified by the "no web app" requirement.

Lightweight alternative worth remembering: pi itself has an SDK/RPC mode (`docs/rpc.md`,
`docs/sdk.md` in the pi package) and could back an island chat with zero extra apps. Choose it
if OpenMausBot's bots/memory/computer features are not wanted. OpenMausBot wins when the point
is the Grok-Bot feel (a roster of named bots, per-bot memory, computers, approvals).

Caveats to accept for OpenMausBot on this machine:

- Ubuntu build is a beta (.deb / AppImage; Arch user would run the AppImage or build from
  source). Wayland: local-computer control is **fail-closed** (their issue #345) — irrelevant
  for chat/status; cloud computer or local VM remain available for hands.
- Harness API is undocumented in the repo's public docs → treat the endpoint set as a
  version-pinned integration (Phase 0 task: enumerate `server/routes/` and record the chat,
  events, approvals endpoints in this repo).

## 2. Architecture

```
┌──────────────────────────────────────────────────────────┐
│ OpenMausBot (tray, no window)                            │
│   Electron shell ── embedded harness server 127.0.0.1:8799│
│   - bots, turns, memory, approvals, computers             │
└───────────────┬──────────────────────────────────────────┘
                │ HTTP commands (send turn, approve, deny)
                │ SSE event stream (tokens, tool calls, state)
┌───────────────▼──────────────────────────────────────────┐
│ Adapter: bloub-island-openmaus (tiny bridge process)      │
│   - SSE → protocol v1 events → unix socket                │
│   - socket commands → HTTP                                │
│   (keeps harness-specific code out of QML, per adapters.md)│
└───────────────┬──────────────────────────────────────────┘
                │ bidirectional NDJSON on $XDG_RUNTIME_DIR/bloub-island.sock
┌───────────────▼──────────────────────────────────────────┐
│ Quickshell: Agents.qml → DiAgent.qml → BloubAvatar.qml    │
│   collapsed: mascot + clock · hover: status               │
│   click / hotkey: chat panel (messages + input)           │
└──────────────────────────────────────────────────────────┘
```

The island never knows OpenMausBot exists. A pi adapter, an AG-UI adapter or a future harness
adapter speaks the same socket (see `adapters.md`).

## 3. Protocol v1 becomes bidirectional

Events (adapter → island) stay exactly as in `protocol/v1.schema.json`. Commands
(island → adapter) are new lines on the same socket:

```json
{"v":1,"command":"send","sessionId":"maus:thread-1","text":"ringkas inbox gue"}
{"v":1,"command":"approve","sessionId":"maus:thread-1","approvalId":"ap_42","decision":"once"}
{"v":1,"command":"cancel","sessionId":"maus:thread-1"}
{"v":1,"command":"ping"}
```

- `send` — submit a turn (chat message).
- `approve` — `decision`: `once` | `always` | `deny` (mirrors Grok Bot's approval model).
- `cancel` — interrupt the running turn if the harness supports it.
- `ping` — liveness/keepalive.

Adapter replies (optional, same socket): `{"v":1,"event":"ack","commandId":…,"ok":true}`.
Unknown commands must be ignored by adapters; unknown events by the island. Schema files:
`protocol/v1.schema.json` (events) + a `commands` section to be added in Phase 1
(backwards compatible: commands only appear when a capability is advertised).

Handshake: on connect the adapter sends `{"v":1,"event":"hello","capabilities":{"commands":["send","approve","cancel"],"harness":"openmausbot"}}`.

## 4. Island UX

| State | Pill (32 px) | Expanded |
|---|---|---|
| No session | clock (unchanged) | calendar overlay (unchanged) |
| Agent idle with a bot selected | mascot `idle` + clock | status line |
| Streaming | mascot `thinking` + "…" | last message snippet |
| Tool | mascot `play`/`orbit` + tool name | live tool line |
| Waiting (approval/question) | mascot `alert`, **always-win** | approval card with Allow once / Always / Deny |
| Done | mascot `burst` → `idle` 3 s | "done ✓" |
| Error | `exclaim`, red | error text |

- **Trigger**: click the mascot, or a Hyprland keybind (`Agents.toggleChat()`).
- **Chat panel**: overlay anchored under the pill (like the calendar), `ListView` of messages,
  `TextInput` at the bottom, `Esc` closes, `Enter` sends, `Shift+Enter` newline.
- **Focus**: the panel needs keyboard focus — use `WlrKeyboardFocus.OnDemand` (or
  `GlobalFocusGrab`, already present as `services/GlobalFocusGrab.qml`); `DiSession` is the
  existing precedent for `forceActiveFocus()` on a provider.
- **App stays in tray**: start OpenMausBot at login with its window hidden; the island is the
  only surface. If the tray app is not running, the island hides the chat entry point.
- **Attachment/voice**: out of scope for v1 (voice memos/drafts are Grok Bot polish, not core).

## 5. SSE in QML, without blocking the shell

- Preferred: the adapter process owns the SSE connection (`fetch`/`undici` in Node), so QML
  only reads a local socket. QML never talks HTTP to a backend.
- If a direct fallback is ever needed: `Quickshell.Io.Process` running `curl -N <url>` with a
  `SplitParser { splitMarker: "\n\n" }` and `stdout` data parsing — never `XMLHttpRequest`
  streaming.
- Reconnect with exponential backoff inside the adapter; coalesce token deltas (≤10 Hz) exactly
  like the pi extension (`agent-bridge.md` § 2).

## 6. Phase plan changes

- **Phase 1 (revised):** bridge + socket + `Agents.qml` + `DiAgent` status (as before), **plus**
  the bidirectional command schema and an adapter skeleton for one backend.
- **Phase 3b (new): chat in island.** Backend = OpenMausBot harness (or pi RPC as the light
  path). Deliverables: chat overlay, message list, input focus, approval card, send/approve
  commands, tray autostart config, `docs/` for the adapter.
- Everything else (mascot port, theme, multi-session) unchanged.

## 7. Acceptance (island chat)

- [ ] Send a message from the island with no OpenMausBot window ever opened; reply streams in.
- [ ] Tool calls and phase changes move the mascot without stealing focus.
- [ ] An approval can be answered from the island (or falls back to opening the app).
- [ ] `Esc` returns focus to the previously focused window; no stuck keyboard grab.
- [ ] Killing OpenMausBot: island degrades to clock + "offline" badge, no QML errors.
- [ ] Killing Quickshell: adapter reconnects and the harness keeps running (app untouched).

## 8. Open questions

1. Exact harness endpoints (turns, SSE path, approvals) — enumerate `server/routes/` in
   `milind-soni/OpenMausBot` and pin the app version; record the findings here.
2. Tray-only launch on Hyprland: does the Electron app start hidden reliably under uwsm?
   (their `docs/verification/startup-tray.md` suggests yes; verify on our machine.)
3. Input method conflict (fcitx5) with layer-shell focus — test early with the launcher panel.
4. Multi-bot roster in the island: one hero bot v1; roster switch (`@`-mention equivalent) later.
5. If OpenMausBot's API proves too unstable, fall back to the pi RPC path and keep the same UX.
