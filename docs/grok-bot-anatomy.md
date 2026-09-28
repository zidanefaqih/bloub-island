# Grok Bot anatomy — what's public, and how to get close

Researched 2026-09-28. Grok Bot itself is a closed product (xAI/SpaceXAI + Cursor, launched
2026-08-11), but there are four usable public layers: official docs, official open-source
crates that ship inside the same monorepo, unofficial reconstructions, and a field-notes
playbook from the launch team. Everything below was verified against those sources.

## 1. Official product architecture (docs.x.ai/grok-bot)

| Subsystem | Facts from official docs |
|---|---|
| The computer | **One persistent cloud computer per account**, shared by *all* your Bots: browser, filesystem (`/workspace`), terminal. Shared cookies/sessions/CLI credentials. Isolation is **between users**, not between Bots. One Bot = one *screen*; screens are work surfaces, not security boundaries. Cloud runs in Cursor's cloud. |
| Agent UX | Bots have names, jobs, per-Bot memory that compounds; messaging (text/dictation/voice chat), voice memos, drafts-you-approve, one Bot reachable from desktop + mobile. |
| Tools | **Connectors** installed from a Marketplace (account-wide plugins) preferred where available; **computer use** for everything else (clicking sites like a person). |
| Skills | Learned by **demonstration** — record up to 10 minutes of visible computer interaction (no mic), the Bot drafts a skill: when to use, inputs, sequence, validation, output, approval rules. `/` references skills, `@` references Bots/groups/routines/connectors. |
| Routines | Schedule or event triggers that run a skill as one Bot. |
| Coordination | Bots run in parallel, message each other, share context in group chats, hand off ownership. |
| Approvals | Per-action approval UI: Allow once / Always allow (saves a matching rule) / Deny. **Auto Review**: model-based rules with *Ask first* (wins ties) and *Allow automatically*; team rules can be admin-locked; personal rules are per desktop and sync to the Bot computer. |
| Human handoff | Bot asks you to take over for passwords/passkeys, 2FA, CAPTCHA, payment/identity checks. Secure secret requests are masked and never enter the conversation. |
| Lifecycle | Update / Recover / Reset with snapshots; files, sessions and sign-ins survive normal updates. Work continues while your laptop is closed. |
| Economics | Official case study: support tickets +175 % without new headcount at **~$0.20–0.30 per resolution** (<https://x.ai/news/grok-bot-customer-support>). |

## 2. Official open source that contains Grok Bot internals

### `xai-org/grok-build` — 27 k★, Rust, Apache-2.0

<https://github.com/xai-org/grok-build> — *"SpaceXAI's coding agent harness and TUI"*,
**synced periodically from the SpaceXAI monorepo**. It is the `grok` CLI, not the Grok Bot
product, but the sync drags in shared `crates/common/` used by Grok Bot as well:

| Crate | What it reveals |
|---|---|
| `xai-tool-protocol` | **`src/bot_relay.rs` (79 KB)** and `src/frames.rs` (92 KB): the **BotRelay protocol** (JSON-RPC-style frames, handshake, capabilities, registration, session events, turn hooks, error codes). Plus generated clients: `generated/swift/BotRelayProtocol.swift` (547 KB), `generated/kotlin/BotRelaySchema.kt`, `BotRelayCommands.kt`, `BotRelay.kt`. **This is the client↔bot relay contract, in public.** |
| `xai-tool-protocol/fixtures/bot_relay/` | Golden JSON fixtures incl. `error_box_migrating`, `error_box_recreating`, `error_box_unavailable`, `error_command_rejected*` — the cloud-computer lifecycle is encoded here. |
| `xai-computer-hub-core` / `-sdk` / `-mcp-adapter` | The **computer-use hub**: browser/desktop control surface plus an MCP adapter. |
| `xai-tool-runtime`, `xai-tool-types` | Tool execution runtime and wire types. |
| `xai-message-delivery-core` | Message delivery (bot↔bot / bot↔user plumbing). |
| `xai-interjection-core` | Steering: injecting user input mid-run. |
| `xai-grok-compaction` | Context compaction. |
| `crates/codegen/xai-grok-workspace` (`permission/manager/mod.rs`, 349 KB) | **Permission/approval engine.** |
| `crates/codegen/xai-grok-shell` (`agent/relay.rs`, `relay/`, `leader/protocol.rs`) | Agent runtime + relay + leader/worker protocol. |
| `crates/codegen/xai-grok-pager`, `xai-grok-tools` | TUI, terminal/file/search tools. |

Platform signal: generated **Swift and Kotlin** relay clients → Grok Bot's mobile apps are
native. The desktop app is **Electron** (see § 3).

### `xai-org/grok-prompts` — AGPL-3.0

Prompts for the Grok chat assistant and the `@grok` bot on X — *not* the Grok Bot product,
but the public sample of xAI's prompt style.

## 3. Unofficial reconstructions (research only — check licenses)

- **`b-nnett/grok-bot-0.18-reconstructed`** — 3.5 k★, **no license**. Source-oriented
  reconstruction of the shipped **macOS 0.18.0 Electron app**: readable TypeScript for the
  *Electron, host, coordinator, local-execution, protocol and renderer* boundaries, plus a
  bootstrap that re-downloads the pinned official installer and rebuilds it; adds a local
  Docker sandbox instead of the remote box and an inference router for other models. Notes
  the shipped frontend was minified-only, so the renderer stays the original binary's.
- **`2217173240/grok-bot-box-image`** — black-box reconstruction of the **cloud computer
  base image** (`grok-box-base:arm64`): `debian:trixie` + full desktop (Xvfb, xfwm4, x11vnc,
  **noVNC**, Chromium, CJK fonts) + toolchains (Go, Rust, Python, bun, uv, gh, pinned
  Playwright) in ~31 build steps. Inside the container: `box-init` (entry), `start-desktop.sh`
  (per-screen process supervision table), `start-window`/`stop-window` (secondary screens +
  owner tokens), `box-chrome` (**CDP on 9222+N**), `sand-window-router.mjs` (display routing),
  `session-sync.mjs` (login-state sync daemon). Persistent volume `/home/box/sand-data` keeps
  Chromium profile + sign-ins across container replacement. This is the closest thing to a
  recipe for Grok Bot's "computer".
- Other community clones worth reading: `GrokNode` (TypeScript reconstruction),
  `OpenMausBot`, `rakazo`, `Open-GrokBot`, `RedSky-Bot`, `TecAdRiseBot`, `hermes-bot-kit`.

## 4. Field notes from the launch team

**`unicodef1wn/grokbot-field-notes`** — MIT. Three xAI Grok Bot engineers built and launched a
product from an empty repo in 72 hours, live, on their own platform. The repo packages:
`AGENTS.md` (house rules you drop into a repo), `ANTIPATTERNS.md` (40 things that broke on
air), `agents/` (verification, orchestration, skills/routines, prompts), `roster/` (69 bot
role descriptions), `playbooks/` (9 role workshops), `reference/ECONOMICS.md` +
`reference/PRODUCT.md`, and 24-page + 14-page PDF guides. This is the best free look at how
teams actually operate Grok-Bot-style fleets.

## 5. A credible "Grok Bot-like" stack (if you want to build, not use)

Minimum viable clone — seven subsystems, each with a public reference:

| # | Subsystem | Reference / off-the-shelf |
|---|---|---|
| 1 | Chat client (desktop + mobile) | Electron/Tauri app; OpenMausBot's Telegram-style shell |
| 2 | Relay protocol | model it on `bot_relay.rs` + fixtures (JSON-RPC frames, session events); or adopt **AG-UI** |
| 3 | Agent runtime | pi (extensions/MCP), Claude/Codex CLIs, or your own loop; `xai-grok-shell` for shape |
| 4 | Tool runtime + approvals | `xai-tool-runtime` shape; approval UI rules from docs; `permission/manager` for policy design |
| 5 | The computer | Docker + Xvfb/xfwm4/x11vnc/noVNC + Chromium (copy `grok-bot-box-image`), per-bot screens, CDP `9222+N`, persistent volume; or rent it (Boat API, E2B; OpenMausBot uses Boat) |
| 6 | Connectors | MCP + Composio (500+ apps, used by OpenMausBot) |
| 7 | Skills + routines | record→skill format from the docs; cron/event scheduler |

Adjacent usable products: **OpenMausBot** (local harness server `127.0.0.1:8799`, one SSE
stream, per-thread NDJSON, drivers for Claude/Codex/Grok Build, Boat for cloud computers,
Composio for apps, Apache-2.0) and **OpenBot/CopilotKit** (AG-UI platform with a governance
gateway, MIT, Docker + Postgres). Neither is Grok Bot, both get you the shape.

## 6. Where bloub-island fits

The island is a **client UI component**, not the product. If a local Grok-Bot-like ever gets
built, this repo's protocol v1 is a miniature relay: `Agents.qml` folds one canonical event
stream into state, exactly like Grok Bot's desktop app folds BotRelay frames. Keep the island
harness-agnostic (see `adapters.md`) and it becomes the Linux desktop face of whatever
runtime you plug underneath — pi today, a harness server tomorrow.

## 7. Practical cautions

- `xai-org/grok-build`: Apache-2.0 — studying/reusing with attribution is fine.
- `b-nnett/...reconstructed` and `grok-bot-box-image`: **no license / unclear license** —
  research reference only; do not copy code or ship their artifacts.
- The Grok Bot desktop/mobile binaries are proprietary; reconstructions re-download the
  official installer and are not redistributable.
- Cloud computers with shared browser sessions and credentials are a security boundary
  problem: Grok Bot's answer is per-user isolation + approvals + Auto Review. Any clone that
  skips that is a toy; treat credentials, 2FA handoff and irreversible actions as first-class
  from day one.
