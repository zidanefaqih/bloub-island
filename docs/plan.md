# Plan

## 1. Goal

Make the `end4-pC` bar's dynamic island reflect the live state of **pi coding agent**
sessions, using the **bloub** mascot (the x.ai blob avatar) as the agent's face.

Success looks like:

- Start `pi` in any terminal → the island notices within a second and expands with the
  mascot "thinking".
- While a tool runs, the island says what tool and keeps animating; while pi streams, it
  keeps a live timer.
- When pi needs approval or an answer, the island insists (always-win) and shows `alert`.
- When the turn settles, a `burst` plays, then everything collapses back to normal.
- With no pi running, the bar is byte-for-byte as before.
- The mascot follows the theme (light/dark) with no hardcoded colors.

## 2. Non-goals (v1)

- No control of pi from the island (no "approve" button in v1; clicking focuses the
  terminal instead). Approval UI stays in the TUI.
- v1 ships the **pi adapter only**, but the island must stay harness-agnostic: protocol v1
  carries an optional `agent` field and the adapter contract is documented in
  [`adapters.md`](./adapters.md). Adding a harness must never require touching the island.
- No bloub customization studio (shapes/colors/expressions editor). Only the measured
  default silhouette + a few states relevant to agent status.
- No Windows/macOS. Linux/Quickshell only. No `Quickshell.Web` (not available).

## 3. Architecture

```
┌───────────────────────────┐
│ Harnesses (layer 1)       │  pi today; OpenMausBot/OpenBot/Claude Code/Codex later
│  adapter: bloub-island.ts │  hooks: session_*/agent_*/turn_*/message_*/
│  → NDJSON over unix sock  │         tool_execution_*/ui_prompt_*/compact_*
└───────────┬───────────────┘
            │ $XDG_RUNTIME_DIR/bloub-island.sock
┌───────────▼───────────────┐
│ services/Agents.qml       │  Quickshell.Io SocketServer + SplitParser
│  - sessions{} registry    │  phases, tool names, timers, staleness
│  - busiest() / waiting()  │  derived: activeProvider data, badge count
└───────────┬───────────────┘
            │ QML bindings
┌───────────▼───────────────┐
│ modules/ii/bar/DiAgent.qml│  pill content (32 px high)
│  ├─ BloubAvatar.qml       │  ported engine, sampler driven by Timer
│  ├─ label / elapsed       │
│  └─ click/hover behavior  │
└───────────┬───────────────┘
            │ provider entry in contentProviders
┌───────────▼───────────────┐
│ DynamicIsland.qml         │  expands pill, badges, always-win when waiting
└───────────────────────────┘
```

Any adapter can write the same protocol — an SSE client for OpenMausBot's harness server
(`127.0.0.1:8799`), an AG-UI client for OpenBot, or a plain hook script. The island (layer 3)
never learns about any harness; see [`adapters.md`](./adapters.md).

### Why a socket, not a file/poll

- Event rates are bursty (`message_update` streams per chunk); a socket gives us push with
  no polling and no partial-read races.
- Quickshell 0.2.1 ships `Quickshell.Io.SocketServer` / `Socket` / `SplitParser` (verified
  against `quickshell-io.qmltypes`), so this is zero extra runtime dependencies.
- Fallback (if socket proves flaky): atomic JSON state file + `FileView { watchChanges }`.
  Keep this in your pocket, not in v1.

## 4. Phases, deliverables, acceptance

### Phase 1 — bridge PoC (no mascot)

Deliverables:

- `bloub-island.ts` pi extension (global: `~/.pi/agent/extensions/` or project-local).
- `services/Agents.qml` singleton: `SocketServer`, NDJSON line parsing, session registry,
  staleness/timeouts.
- `modules/ii/bar/DiAgent.qml`: `[icon] [label] [elapsed]`, following `DiOsd.qml` shape.
- `DynamicIsland.qml`: one new provider entry + `Component` + `iconForProviderId` case.

Acceptance:

- `pi` started in a terminal → island shows "thinking" within 1 s.
- Running a bash tool → label switches to `bash`; ending it → back to idle/streaming.
- `ctx.ui` prompt (approval/select) → phase `waiting`, island does not get replaced by a
  notification provider (always-win path verified).
- `agent_settled` → `done` for ~3 s → collapse; if no other provider is active, island
  returns to exact prior width.
- Two concurrent pi sessions → island shows the busiest one, badge count shows `+1`.
- Kill pi with `SIGKILL` (no shutdown event) → island clears within 10 s (staleness rule).

### Phase 2 — mascot MVP

Deliverables:

- Ported engine under `modules/common/bloub/` (see `docs/bloub-port.md` for the file set).
- `modules/common/widgets/BloubAvatar.qml`: `sample(t)` on a 30–60 fps `Timer`, renders
  via `Shape` (preferred) or `Canvas`, colors from `Appearance`.
- Wire `DiAgent.qml` to it. States: `idle`, `thinking`, `alert`, `burst`.

Acceptance:

- Frozen frames (`frozenAt` equivalent) of each ported state match upstream screenshots
  from `https://bloub.vercel.app/#planche` (antialiasing differences OK).
- 32 px rendering stays readable: silhouette + eyes legible, no shimmer.
- CPU: quickshell CPU delta while idle ≤2 % on the dev machine; while animating ≤5 %.

### Phase 3 — polish

- All relevant states (`wide`, `notify`, `exclaim`, `play`, `orbit`, `sleep`, `comet`,
  `swirl` as feasible at 32 px).
- Hover-expanded island (~80 px mascot + tool name + last message snippet).
- Click → focus the pi terminal window (extension reports Hyprland window address; fallback
  `hyprctl dispatch focuswindow pid:<pid>`).
- `Config.options` toggle to disable the widget entirely.
- Light/dark theme pass; accent color option.

### Phase 4 — ship

- Sync into live config, document the extension in the end4-pC README.
- Optional: upstream PR to `zidanefaqih/end4-pC`.
- Tag v0.1.0; record the extension version + schema version compatibility.

## 5. Risks & mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Porting the TS engine is bigger than it looks | Phase 2 stalls | Strict minimal port set; `eyefit` stubbed to 0 (no custom shapes); validate against upstream per state |
| 32 px is too small for state detail | Mascot unreadable | Restricted state subset in pill; full detail in hover-expanded view |
| `Shape` re-tessellates per frame (CPU) | Frame drops | Measure; fall back to `Canvas` (raster but trivial at 32 px); cap 30 fps |
| pi extension API changes | Bridge breaks | Protocol versioned; extension isolated in one file; unknown events ignored |
| Socket races/partial lines | UI glitches | `SplitParser` + tolerate incomplete JSON; never let a parse error kill the service |
| Multiple sessions confuse the UI | Wrong state shown | Deterministic priority (waiting > tool > streaming > done), badge for the rest |
| Theme bleed (black blob on dark bar) | Invisible mascot | Body = `colOnLayer0`, eyes are true holes (`OddEvenFill`), never hardcode black |

## 6. Open questions

1. `Shape` vs `Canvas` — decide by measurement in Phase 2, not by preference.
2. Should the mascot also appear when pi is *not* running (e.g. `sleep` face as an idle
   decoration)? Default: no; opt-in config flag later.
3. Where exactly to place the provider relative to `notification`/`battery` priority —
   proposal: `agent` in `alwaysWinIds` **only** while waiting; otherwise lowest priority.
4. Should the extension ship global (`~/.pi/agent/extensions/`) or project-local? Global
   (every device benefits); document env var to disable: `BLOUB_ISLAND=0`.

## 7. Effort estimate

| Phase | Effort |
|---|---|
| 1 — bridge PoC | 1 session (~2–3 h) |
| 2 — mascot MVP | 1–2 sessions |
| 3 — polish | 1–2 sessions |
| 4 — ship | <1 session |

## 8. Resulting repo layout (target)

```
end4-pC/
  modules/common/bloub/            # ported engine (framework-free JS + QML)
  modules/common/widgets/BloubAvatar.qml
  modules/ii/bar/DiAgent.qml
  services/Agents.qml
~/.pi/agent/extensions/bloub-island.ts
bloub-island/                      # this repo: spec now, dev harness later
```
