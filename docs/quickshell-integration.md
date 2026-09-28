# Quickshell / end4-pC integration

Target config: [`zidanefaqih/end4-pC`](https://github.com/zidanefaqih/end4-pC).
Repo checkout: `~/Projects/end4-pC`. Live config: `~/.config/quickshell/end4-pC`
(**plain copy, not a symlink** — never edit it as the source of truth).

Environment: Arch Linux, Hyprland/uwsm, Quickshell **0.2.1** (`quickshell-git` AUR).

## 1. How the island works today

`modules/ii/bar/DynamicIsland.qml` (460 lines) is a provider system:

| Anchor | Line (approx) | Meaning |
|---|---|---|
| `pillHeight: 32` | 18 | pill height; mascot must fit ~24 px |
| width properties | 20–27 | per-provider widths, e.g. `notificationWidth: 220` |
| `contentProviders` | 194 | array `{ id, active, component, width }` |
| `alwaysWinIds` | 204 | providers that override everything (`session`, `notification`, `battery`, `osd`) |
| `activeOthers` / `activeProvider` | 206–216 | priority resolution |
| `badgeProviders` | 219 | other active providers become `+N` badges |
| `iconForProviderId` | 223 | maps provider id → Material Symbol for badges |
| `implicitWidth` + `Behavior` | 241–256 | animated expansion (350 ms, expressive bezier) |
| `Loader` + `Component`s | 300–460 | one `Component` per provider, `DiIdle`…`DiSession` |

Provider components follow one shape:

```qml
RowLayout {
    id: diOsdRoot
    required property Item di        // access to DynamicIsland (root)
    anchors { fill: parent; leftMargin: root.isMaterial ? 4 : 8; rightMargin: 10 }
    spacing: 6
    // icon + center spacer + text
}
```

`DynamicIsland.qml` is instantiated from the panel families; new files in
`modules/ii/bar/` need no registration. `services/*.qml` with `pragma Singleton` are
auto-discovered through `import qs.services` (this config has **no `qmldir` files** —
Quickshell directory imports do the work).

## 2. Changes to `DynamicIsland.qml`

### a) New width property (near line 27)

```qml
readonly property real agentWidth: Agents.hero ? Math.min(220, 44 + agentLabel.implicitWidth) : 132
readonly property real agentExpandedWidth: 260
```

(Concrete numbers are tuning; the point is a reactive width like the others.)

### b) Provider entry (inside `contentProviders`, line 194)

```qml
{ id: "agent", active: Agents.hero !== null, component: agentComponent, width: root.agentWidth },
```

Place it after `notification` so an agent never hides a notification — the always-win rules
below handle the "needs attention" case.

### c) Always-win while waiting (line 204)

`alwaysWinIds` is currently a static array. Make it reactive so that *waiting for user* is
never swallowed:

```qml
readonly property var alwaysWinIds: ["session", "notification", "battery", "osd"]
    .concat(Agents.waiting ? ["agent"] : [])
```

`waiting` = hero phase `waiting` (see `docs/agent-bridge.md` § 4). Everything else about the
agent keeps normal priority, so notifications still win while pi is merely streaming.

### d) Badge icon (line 223)

```qml
case "agent": return "smart_toy"   // or a small monochrome blob glyph
```

### e) Component (after the session component, ~line 337)

```qml
Component {
    id: agentComponent
    DiAgent { di: root }
}
```

## 3. `modules/ii/bar/DiAgent.qml`

```qml
import QtQuick
import QtQuick.Layouts
import qs
import qs.services
import qs.modules.common
import qs.modules.common.widgets
import qs.modules.common.bloub

RowLayout {
    id: diAgentRoot
    required property Item di
    anchors { fill: parent; leftMargin: root.isMaterial ? 2 : 4; rightMargin: 10 }
    spacing: 6

    BloubAvatar {
        id: mascot
        Layout.alignment: Qt.AlignVCenter
        size: 22
        state: Agents.hero?.bloubState ?? "idle"     // see bloub-port.md § 6
        gazeTarget: hoverHandler.hovered ? hoverHandler.point.position : null
    }

    StyledText {
        id: agentLabel
        Layout.alignment: Qt.AlignVCenter
        text: Agents.heroLabel                            // "thinking…", "bash", "waiting", …
        font.pixelSize: Appearance.font.pixelSize.small
        color: Agents.hero?.phase === "error"
            ? Appearance.colors.colError
            : Appearance.colors.colOnLayer0
    }

    Item { Layout.fillWidth: true }                        // right-align elapsed

    StyledText {
        Layout.alignment: Qt.AlignVCenter
        visible: Agents.heroElapsedText !== ""
        text: Agents.heroElapsedText
        font.pixelSize: Appearance.font.pixelSize.small
        font.features: { "tnum": 1 }
        color: Appearance.colors.colOnLayer0
    }

    HoverHandler { id: hoverHandler }

    MouseArea {
        anchors.fill: parent
        cursorShape: Qt.PointingHandCursor
        onClicked: Agents.focusHero()                      // Phase 3
    }
}
```

`Agents.qml` owns all derived presentation state (`hero`, `heroLabel`, `heroElapsedText`,
`waiting`, `bloubState`) so the view stays dumb. Same for the state mapping
(`phase + tool → StateId`) — keep it in one JS function in `Agents.qml`.

## 4. `services/Agents.qml`

Skeleton and transport semantics live in [`agent-bridge.md`](./agent-bridge.md) § 5.
Responsibilities:

1. `SocketServer` + `SplitParser` on `$XDG_RUNTIME_DIR/bloub-island.sock`.
2. Registry of sessions, priority resolution, staleness timers.
3. Derived UI state: `hero`, `heroLabel`, `heroElapsedText`, `waiting`, `count`, `bloubState`.
4. `focusHero()` — Phase 3 (`Hyprland.dispatch("focuswindow", ...)`).
5. Defensive parsing: a malformed line or unknown event must never break the shell.

## 5. Theming

- Mascot body: `colOnLayer0`; holes/paper: `colLayer0` (pill colour).
- Optional accent: body `colPrimary` when phase is `waiting` (draws the eye).
- Error badge/label: `colError`; done: `m3success` if that token exists in `Appearance`.
- Never hardcode colours; follow `Appearance.colors.*` and `Config.options`.

## 6. Dev & test workflow

1. Edit in `~/Projects/end4-pC`.
2. **Preview harness first.** Add `dev/agent-preview/shell.qml` (a floating
   `PanelWindow` with the mascot + buttons to force each phase). Run:

   ```bash
   qs -p ~/Projects/end4-pC/dev/agent-preview
   ```

   Keep it `exclusionMode: Ignore` so it does not reserve space. This avoids running two
   bars at once.

3. Full-config test (stop the running shell first, or accept overlapping bars):

   ```bash
   qs -p ~/Projects/end4-pC
   ```

4. Fake traffic without pi:

   ```bash
   SOC=${XDG_RUNTIME_DIR}/bloub-island.sock
   echo '{"v":1,"event":"agent_start","sessionId":"test","ts":1,"phase":"thinking"}' \
     | socat - UNIX-CONNECT:$SOC
   ```

5. Sync to live once satisfied:

   ```bash
   rsync -a --delete --exclude .git ~/Projects/end4-pC/ ~/.config/quickshell/end4-pC/
   ```

   (Or copy only the touched files — `--delete` removes anything untracked in live.)

## 7. Regression checklist

- [ ] With no pi running: island looks and behaves exactly as before.
- [ ] Notifications still interrupt while pi streams; only `waiting` overrides.
- [ ] Wheel-to-dismiss (`forceIdle`) still works when the agent provider is active.
- [ ] Badge row shows `+N` for a second session; clicking a badge focuses it.
- [ ] Vertical bar mode (`Config.options.bar.vertical`) hides the provider cleanly
      (`visible: !root.vertical` pattern already used by the pill).
- [ ] No QML warnings in the journal when providers switch.
