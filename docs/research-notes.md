# Research notes

Raw findings collected while writing the spec. Versions and facts, not decisions.

## Bloub (the mascot)

- Repo: <https://github.com/jeremy-prt/bloub> — MIT, 1.6 k★ / 201 forks (as of 2026-09-28),
  created 2026-08-15 (4 days after Grok Bot's public launch), last push 2026-08-17.
- Description: *"SVG recreation of the x.ai bot avatar. One shape morphing through 14
  states, measured off the reference video frame by frame."*
- Stack: Vue 3, Vite, TypeScript, Tailwind 4, **no animation library**. Topics: animation,
  avatar, morphing, svg, svg-animation, typescript, vue.
- Homepage / demo: <https://bloub.vercel.app> (the vercel.app URL *is* the project's
  homepage — the repo is the proper source).
- Dev: `pnpm dev` → <http://localhost:5190>; `pnpm test` (vitest); `pnpm build`
  (`vue-tsc --noEmit && vite build`). No ESLint/Prettier — `pnpm build` is the gate.
- Deep links: `#planche` (14 states frozen side by side), `#etat=<id>&stop` (one state
  paused). Used as the validation board for the QML port.
- Upstream docs worth reading before porting: `docs/architecture.md`,
  `docs/measurements.md`, `docs/intro.md`, `docs/interface.md`, `docs/export.md`,
  `docs/i18n.md`. Profiles can be regenerated with `tools/extract-profiles.py`.
- Engine API: `new BotEngine(R, state, shapeRadii, expression)`;
  `sample(now): BotFrame`, `setState`, `reset`, `setShape`, `setExpression`, `setLook`,
  `.state`. `BotFrame.bodyPath` and `eyes[].d` are SVG path strings; `eyes[].matrix` is an
  SVG transform. Eyes are mask holes; body is backed by a paper-coloured path for
  occlusion; `radiusAtAngle` keeps eyes on the silhouette; `eyefit` is a pre-solved table
  for custom shapes (solve-once-at-import, never per frame).
- Community ports/derivatives seen on GitHub: `ShunquanWang/bloub-react`,
  `SomeRandoLameo/react-bloub`, `arinagrawal05/reactive_bloub_flutter`,
  `atq-ren/esp32-robot-face` (round AMOLED), `Yuuhann1999/dsh-bloub-mood` (favicon plugin),
  `Eyadkelleh/Grok_bot` (SVG avatar studio). **No QML/Quickshell port exists** as of
  2026-09-28 — this project would be the first.

## Grok Bot context (why the mascot exists)

- **Grok Bot** launched **2026-08-11** by xAI ("SpaceXAI") with Cursor: always-on AI
  teammates, each with one persistent cloud computer (browser, files, terminal), working
  inside your tools. Not to be confused with grok.com chat or Grok Imagine.
- Included with SuperGrok (Plus/Heavy) and Cursor Pro/Teams; limited free trial. Desktop
  app macOS/Windows (no official Linux), mobile iOS 18+/Android 9+.
- Related: **Grok Build** (terminal coding agent, open source at `xai-org/grok-build`),
  `xai-org/grok-prompts` (official prompts for Grok chat + `@grok`), community catalogues
  `RongleCat/awesome-grok-bot`, `kydlikebtc/awesome-grokbot`, plus many OSS alternatives
  (OpenMausBot, rakazo, Open-GrokBot, RedSky-Bot, …).

## pi (the harness we bridge)

- Version on dev machine: **0.86.0**, Node v26.10.0.
- Extension docs: `/usr/lib/node_modules/@earendil-works/pi-coding-agent/docs/extensions.md`.
- Extensions are TypeScript modules with `export default function (pi: ExtensionAPI) {}`;
  global discovery at `~/.pi/agent/extensions/*.ts`; hot reload via `/reload`; quick test
  `pi -e ./file.ts`. Node builtins (`node:net`, `node:fs`, …) are available.
- Status-relevant events (with payloads) documented there:
  `agent_start`, `agent_end`, **`agent_settled`** ("Use agent_settled for status
  integrations… Pi will not continue running automatically"), `ui_prompt_start` /
  `ui_prompt_end` (`reason: "ui_prompt"`, `kind: select|confirm|input|editor|custom`,
  `title`), `turn_start`/`turn_end` (`turnIndex`), `message_start`/`message_update`/
  `message_end` (streaming; `message_update` per chunk), `tool_execution_start/update/end`
  (`toolCallId`, `toolName`, `args`, `result`, `isError`), `session_start`/`session_shutdown`,
  `session_before_compact`/`session_compact`/`session_compact_failed`.

## Quickshell

- Version: **0.2.1** (`quickshell-git`, revision 7511545, AUR).
- QML modules installed: `Bluetooth`, `DBusMenu`, `Hyprland`, `I3`, `Io`, `Networking`,
  `Services`, `Wayland`, `Widgets`, `WindowManager`, `X11`, `_Window`. **No `Web` /
  WebEngine** — no webview embedding.
- Verified in `/usr/lib/qt6/qml/Quickshell/Io/quickshell-io.qmltypes`:
  - `SocketServer { active: bool, path: QString, handler: QQmlComponent }`,
    signal `onNewConnection`.
  - `Socket` (prototype `DataStream`): `connected`, `path`, `write(data)`, `flush()`.
  - `SplitParser` (prototype `DataStreamParser`): `splitMarker`, signal `read(QString)`.
  - `DataStream`: `parser: DataStreamParser`.
  - `FileView` also exists (`watchChanges`, `atomicWrites`) — the fallback transport.
- Quickshell supports directory-based QML module imports (no `qmldir` needed).

## end4-pC (the host config)

- Repo `~/Projects/end4-pC` (branch seen: `feat/dnd-suppress-sound`, ahead 1 of `main`),
  public on GitHub. Live copy `~/.config/quickshell/end4-pC` is **not** a git repo/symlink.
- `modules/ii/bar/DynamicIsland.qml` (460 lines) is the provider host: `contentProviders`
  array (line ~194), `alwaysWinIds` (~204), `badgeProviders` (~219),
  `iconForProviderId` (~223), provider `Component`s (~312–460), pill height 32 (line 18),
  350 ms expressive expansion animation.
- Existing provider components to copy patterns from: `DiIdle.qml`, `DiMedia.qml`,
  `DiOsd.qml` (50 lines), `DiTimers.qml`, `DiNotifs.qml`, `DiSession.qml`.
- Services are singletons in `services/` (`pragma Singleton`, e.g. `TimerService.qml`);
  `import qs.services`.

## Adjacent multi-agent projects (evaluated 2026-09-28)

Considered as "do we need a native app first?" — answer: no, but their event streams are
adapter targets.

- **OpenMausBot** — <https://github.com/milind-soni/OpenMausBot>: 3.6 k★, Apache-2.0,
  Electron chat app (macOS/Windows/Ubuntu) driving Claude/Codex/Grok CLIs locally. Its README
  describes a **harness server on 127.0.0.1:8799** exposing HTTP commands + **one SSE event
  stream**; drivers per provider (Claude, Codex, Grok Build via stream-JSON / JSON-RPC / ACP);
  evently logs as per-thread **NDJSON** under `~/.openmausbot`; stdio MCP server for external
  clients. Its README's architecture table (`server/drivers/`, `server/harness/`,
  `server/index.ts`, `server/tts/`) is the canonical-event-stream pattern our adapter layer
  converges on. Ubuntu **Wayland host control disabled** (their issue #345) — irrelevant for
  status consumption.
- **OpenBot (CopilotKit)** — <https://github.com/CopilotKit/openbot>: 5.6 k★, MIT,
  TypeScript, alpha. Self-hosted agent platform (Docker Compose + PostgreSQL + Bun +
  CopilotKit Intelligence); every Bot is any endpoint speaking **AG-UI**
  (<https://github.com/ag-ui-protocol/ag-ui>), ~16 standard event types (run/step/text/tool
  lifecycle, state snapshots/deltas), transport-agnostic (SSE/WebSocket/webhooks); tool calls
  go through a governance gateway. Heavy to run; good adapter target, not a dependency.
- Consequence recorded in [`adapters.md`](./adapters.md): the island is layer 3 and stays
  harness-agnostic; protocol v1 gained optional `agent` + `source` fields.

## Reproduce the verification

```bash
# quickshell modules + socket API
ls /usr/lib/qt6/qml/Quickshell/
grep -n 'SocketServer' /usr/lib/qt6/qml/Quickshell/Io/quickshell-io.qmltypes

# bloub facts
gh api repos/jeremy-prt/bloub --jq '{description,license:.license.spdx_id,stars:.stargazers_count}'
gh api repos/jeremy-prt/bloub/contents/src/bot --jq '.[] | "\(.size)\t\(.name)"'

# pi events
grep -n 'agent_settled' /usr/lib/node_modules/@earendil-works/pi-coding-agent/docs/extensions.md

# island anchors
grep -n 'contentProviders\|alwaysWinIds\|iconForProviderId' \
  ~/Projects/end4-pC/modules/ii/bar/DynamicIsland.qml
```
