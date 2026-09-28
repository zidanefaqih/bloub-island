# bloub-island

A dynamic-island widget for [Quickshell](https://quickshell.org) (built for
[end4-pC](https://github.com/zidanefaqih/end4-pC)) that shows what your
[pi coding agent](https://github.com/earendil-works/pi) is doing — with the x.ai
blob avatar (**bloub**) as its face.

> **Status: Phase 0 — planning only.** No code yet. This repo currently contains the
> full specification so any agent (on any device) can pick the work up cold. Start at
> [`AGENTS.md`](./AGENTS.md), then [`docs/plan.md`](./docs/plan.md).

```
pi TUI session
   │  hooks: agent_start / tool_execution_* / ui_prompt_* / agent_settled
   ▼
~/.pi/agent/extensions/bloub-island.ts      (pi extension, NDJSON writer)
   │  unix socket  $XDG_RUNTIME_DIR/bloub-island.sock
   ▼
services/Agents.qml                          (Quickshell singleton, SocketServer)
   ▼
modules/ii/bar/DynamicIsland.qml  →  DiAgent.qml  →  BloubAvatar.qml
     (extends the existing provider list, pill height 32)
```

## What it looks like

| pi state | bloub state | island |
|---|---|---|
| no session | — | normal collapsed island |
| `agent_start` | `thinking` | expands, "thinking…" |
| streaming | `idle` + gaze | live elapsed/token text |
| tool running | `play` / `orbit` | tool name + spinner |
| waiting for you | `alert` | **always visible**, never swallowed by other providers |
| `agent_settled` | `burst` → `idle` | "done ✓", collapse after ~3 s |
| error | `exclaim` | red badge |
| compacting | `swirl` | "compacting…" |
| long idle | `sleep` | collapse, mascot keeps breathing |

Full tables, specs and file-level instructions: [`docs/`](./docs).

## Docs index

| File | Contents |
|---|---|
| [`AGENTS.md`](./AGENTS.md) | Agent onboarding: environment, rules, contract, next tasks |
| [`docs/plan.md`](./docs/plan.md) | Goals, architecture, phases, acceptance criteria, risks |
| [`docs/bloub-port.md`](./docs/bloub-port.md) | bloub research + how to port its engine to QML |
| [`docs/agent-bridge.md`](./docs/agent-bridge.md) | pi extension + unix-socket protocol spec |
| [`docs/adapters.md`](./docs/adapters.md) | Multi-harness design: how to support other agents without building a native app |
| [`docs/grok-bot-anatomy.md`](./docs/grok-bot-anatomy.md) | How Grok Bot is built, what's open source, how to get close |
| [`docs/island-chat.md`](./docs/island-chat.md) | Goal: chat with the bot from the island; app in tray (OpenMausBot backend) |
| [`docs/openmausbot-notes.md`](./docs/openmausbot-notes.md) | Installed on this machine: AppImage, Wayland quirk, tray, verified API |
| [`docs/quickshell-integration.md`](./docs/quickshell-integration.md) | Exact end4-pC integration points |
| [`docs/research-notes.md`](./docs/research-notes.md) | Raw findings, links, versions, file sizes |
| [`protocol/v1.schema.json`](./protocol/v1.schema.json) | Machine-readable bridge protocol schema |

## Ringkasan (ID)

Widget "dynamic island" di bar end4-pC buat nampilin status agent pi, pakai maskot
bloub (bola x.ai yang bisa morph 14 state). Arsitekturnya: extension pi nulis JSON
via unix socket → singleton Quickshell baca socket → `DiAgent.qml` tampil di pill
island yang sudah ada. Rencana lengkap ada di `docs/`, jangan ngoding sebelum baca
`AGENTS.md`.

## Credits & license

- Bloub engine © [jeremy-prt/bloub](https://github.com/jeremy-prt/bloub) (MIT). Any port
  keeps the MIT license and attribution — see [`docs/bloub-port.md`](./docs/bloub-port.md).
- This repo: MIT.
