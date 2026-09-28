# AGENTS.md — working on bloub-island

Read this file first, then `docs/plan.md`. This repo is a **specification** repo until
Phase 1 lands. Do not invent architecture that contradicts the docs; if you must deviate,
update the doc in the same change.

## What we are building

A Quickshell widget for the `end4-pC` Hyprland rice that renders an "agent dynamic
island" in the bar: when a **pi coding agent** session runs, the island expands and shows
what pi is doing, with the animated x.ai blob mascot (**bloub × jeremy-prt/bloub**) as its
face. pi is the agent harness (it runs these very instructions); its extensions API lets us
listen to session/tool/prompt lifecycle events.

Three moving parts:

1. **pi extension** `bloub-island.ts` — observes pi events, writes NDJSON to a unix socket.
2. **`services/Agents.qml`** — Quickshell singleton, `SocketServer` + per-session registry.
3. **Mascot** — bloub engine ported from TypeScript to framework-free JS, rendered in QML.

The island is **harness-agnostic**: pi is only the first adapter. Any harness (OpenMausBot
SSE, OpenBot AG-UI, Claude Code hooks, …) can write the same protocol; never put
harness-specific logic in QML. See `docs/adapters.md`.

## Environment (this dev machine)

- Arch Linux + Hyprland/uwsm, bash (fish too), Quickshell **0.2.1** (`quickshell-git`, AUR).
- Quickshell QML modules available: `Io`, `Wayland`, `Hyprland`, `Services`, `Widgets`,
  `WindowManager`, `X11`, `I3`, `Networking`, `Bluetooth`, `DBusMenu`, `_Window`.
  **There is no `Quickshell.Web` / WebEngine module** — embedding the bloub web app in a
  webview is not an option on this machine.
- `end4-pC` repo: `~/Projects/end4-pC` (git, the place to edit).
  Live config: `~/.config/quickshell/end4-pC` — **a plain copy, not a symlink**.
  Workflow: edit repo → test with `qs -p ~/Projects/end4-pC` → sync to live afterwards
  (rsync excluding `.git`, or copy files). Never edit the live copy as the source of truth.
- Bloub reference repo checkout (optional, for asset comparison):
  `git clone https://github.com/jeremy-prt/bloub && pnpm install && pnpm dev` → :5190.

## Rules

1. **Engine port stays a pure function of time.** `BotEngine.sample(t)` must remain
   deterministic (no `Date.now()`, no wall clock, no accumulated state outside the
   documented morph history). This is what makes freezing frames, tests and
   screenshot-comparison against upstream possible. See `docs/bloub-port.md`.
2. **Do not modify the live Quickshell config directly.** Changes go to the `end4-pC` repo.
3. **Follow existing end4-pC patterns:** providers in `DynamicIsland.qml` are declared in
   `contentProviders`; singletons live in `services/` with `pragma Singleton`; colors come
   from `Appearance.colors.*`, sizes from `Appearance.font.*`. Never hardcode theme colors.
4. **Respect the bloub license (MIT).** Ported files keep attribution headers and the
   upstream license text; see `docs/bloub-port.md` § License.
5. **Protocol is versioned.** Any change to the socket messages requires a `v` bump and an
   update to `protocol/v1.schema.json` + `docs/agent-bridge.md`.
6. **Throttle `message_update`.** pi emits streaming updates per chunk; the extension must
   coalesce to ≤10 Hz or it will flood the socket.

## The contract you must not break

Transport: NDJSON (one JSON object per line) over `$XDG_RUNTIME_DIR/bloub-island.sock`.

```json
{"v":1,"event":"agent_start","sessionId":"abc123","ts":1759078000.21,"pid":4242,
 "tty":"/dev/pts/3","cwd":"/home/user/proj","title":"proj","phase":"thinking"}
```

- `v`, `event`, `sessionId`, `ts` are always present. Optional fields per event, including
  `agent` (harness id; defaults to `"pi"`) and `source`.
- Full schema: `protocol/v1.schema.json`. Full event→phase mapping: `docs/agent-bridge.md`.
- The QML side must tolerate: unknown fields, unknown `event` values, reconnects, multiple
  concurrent sessions, and a stale socket file.

## Next tasks (ordered)

1. **Phase 1 — bridge PoC.** Write `bloub-island.ts` extension + `services/Agents.qml` +
   minimal `DiAgent.qml` (dot + label, no mascot). Verify: run pi in a terminal, island
   expands with the right text, collapses after `agent_settled`.
2. **Phase 2 — mascot MVP.** Port the bloub engine (see `docs/bloub-port.md` for the exact
   minimal file set) → `modules/common/widgets/BloubAvatar.qml`. Render `idle`, `thinking`,
   `alert`, `burst`. Verify frozen frames against upstream `#planche`.
3. **Phase 3 — polish.** Remaining states, expanded hover view, click-to-focus terminal,
   multi-session badge, theme integration, config toggle.
4. **Phase 3b — chat from the island.** Backend in tray, island as the only UI. Decision and
   design in [`docs/island-chat.md`](./docs/island-chat.md) (OpenMausBot harness +
   bidirectional commands).
5. **Phase 4 — ship.** Sync to live config, README update, optional upstream PR to end4-pC.

## Quality gates

- `qs -p ~/Projects/end4-pC` loads with zero QML warnings related to the new files.
- pi session lifecycle produces exactly the phases in `docs/agent-bridge.md` (manual test).
- With no pi running, the island behaves exactly as before (no regressions in provider
  priority, badges, wheel-to-dismiss).
- Mascot frame parity: frozen states at the same timestamps match upstream screenshots
  (small antialiasing differences acceptable).

## Where to read more

- `docs/plan.md` — phases, deliverables, acceptance criteria, risks.
- `docs/agent-bridge.md` — pi extension skeleton, event catalogue, socket semantics.
- `docs/bloub-port.md` — engine internals, file-by-file port plan, QML rendering options.
- `docs/quickshell-integration.md` — exact lines to touch in `DynamicIsland.qml`, snippets.
- `docs/research-notes.md` — versions, links, measurements, raw reference facts.
