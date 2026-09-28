# OpenMausBot on Arch/Hyprland — install notes and verified API

Verified on the dev machine **2026-09-28**, OpenMausBot **v0.1.89** (release 2026-09-27).
These are machine facts, not guesses; commands used are at the bottom.

## Install what we did

- No AUR package exists (checked AUR RPC). Release assets: x86_64 AppImage (263.6 MB),
  `amd64.deb` (223.5 MB), mac/win builds, plus the **npm package** `openmausbot` (see § 4).
- FUSE2 is not installed, so we used the AppImage's extract mode (no FUSE needed):

```bash
cd ~/Applications
gh release download v0.1.89 --repo milind-soni/OpenMausBot \
  --pattern "OpenMausBot-0.1.89-x86_64.AppImage" --dir .
sha256sum -c <<<"727ddb80c235126185e8049400560add148af9c98bba4367454d2770c5fcc41a  OpenMausBot-0.1.89-x86_64.AppImage"
chmod +x OpenMausBot-0.1.89-x86_64.AppImage
mkdir -p OpenMausBot-0.1.89 && cd OpenMausBot-0.1.89
../OpenMausBot-0.1.89-x86_64.AppImage --appimage-extract && mv squashfs-root app
```

- Installed at `~/Applications/OpenMausBot-0.1.89/app`, launcher at
  `~/.local/share/applications/openmausbot.desktop`, icon deployed to
  `~/.local/share/icons/hicolor/scalable/apps/openmausbot.svg` (the AppImage ships
  `openmausbot.svg`).
- **launcher must pass `--ozone-platform=x11`** — see § 2.

## 1. What works

- App window opens; harness serves **`http://127.0.0.1:8799`**.
- `GET /api/health` → 200, no auth on localhost. Example:

```json
{"app":"openmausbot","pid":53750,"static":true,
 "capabilities":{"guardedMessages":1,"guardedRequests":1,"guardedFullAccess":1,"guardedOnBehalfOf":1}}
```

- `GET /api/events` → **`content-type: text/event-stream`** (SSE), no auth on localhost.
  This is the canonical event stream the island will fold into state.
- `GET /api/bots` → 200.
- Tray: the app registers **StatusNotifierItems** on the session bus (two names owned by
  the app PID), and Quickshell (`org.kde.StatusNotifierHost-…`, `org.kde.StatusNotifierWatcher`,
  both owned by `qs`) is the host — so the tray icon appears in the end4-pC bar.

## 2. Wayland quirk (important)

Native Wayland: the app initializes the Wayland backend, prints ozone/wayland warnings and
**exits 0 without a window or error**. XWayland works:

```bash
~/Applications/OpenMausBot-0.1.89/app/openmausbot --ozone-platform=x11
```

`AppRun` also probes `unshare -Ur true` and appends `--no-sandbox` only if user namespaces are
unavailable (they are available here, so the sandbox stays on).

## 3. Tray + window lifecycle on Linux

From the shipped `electron/main.mjs` (v0.1.89): hide-on-close is **Windows-only** —

```js
win.on("close", (event) => {
  if (process.platform === "win32" && !desktopShutdownStarted && desktopTray) {
    event.preventDefault();
    desktopTray.hide(win);
  }
});
```

So on Linux closing the window quits the app (and the harness). The tray menu still provides
**Open OpenMaus Bot** / **Quit OpenMaus Bot**. Until Linux close-to-tray lands upstream, the
practical "tray-only" move is Hyprland:

```bash
hyprctl dispatch movetoworkspacesilent special:oms,class:com.openmausbot.app
# reopen: click the tray icon, or:
hyprctl dispatch movetoworkspacesilent e+0,class:com.openmausbot.app
```

There is no `--hidden`/`--tray` CLI flag in 0.1.89 (checked `process.argv` handling).

## 4. Headless alternative (best fit for island-only mode)

The release also ships an npm package: `openmausbot@0.1.89`, Apache-2.0, `bin: cli.js`,
`engines.node >= 24`, ~38.5 MB unpacked.

```bash
npm install -g openmausbot
openmausbot            # first run = setup wizard, then starts the server
openmausbot setup      # revisit AI/phone setup, save, exit
openmausbot --no-open  # start without opening a browser
```

- Foreground server (Ctrl-C stops it; saved work persists). It opens the workspace in a browser
  by default; `--no-open` suppresses that.
- This is the cleanest backend for the island: **no Electron window at all**, run it as a
  `systemd --user` service, island talks to `127.0.0.1:8799`.
- Engine drivers are the local CLIs (`claude`, `codex`, `grok`) or an API connection; this
  machine already has `grok` 1.0.40. `/api/cli-candidates` and `/api/cli-test` exist to probe.

## 5. Verified endpoint inventory (island-relevant)

Strings extracted from the shipped `resources/server` bundle (paths are real; auth behaviour
verified only for the ones marked 200):

| Endpoint | Verified | Notes |
|---|---|---|
| `/api/health` | 200, no auth | capabilities, pid |
| `/api/events` | SSE, no auth | **the canonical stream for the adapter** |
| `/api/bots` | 200 | bot roster |
| `/api/groups` | – | group chats |
| `/api/instances` | – | running agent instances; `/api/instances/claude-accounts` |
| `/api/decisions`, `/api/decisions.csv` | – | approvals audit — maps to `approve` command |
| `/api/attachments` (+ `/api/attachments/…`) | – | files |
| `/api/config`, `/api/edition`, `/api/brand` | – | settings/branding |
| `/api/connectors*`, `/api/computers/{boxes,vps}` | – | connectors, cloud computers |
| `/api/internal/threads`, `/api/internal/retry-thread` | – | thread control |
| `/api/internal/ask-bot` | – | **candidate for submitting a turn** (to pin) |
| `/api/auth/*` | – | session/pair/OTP; localhost single-user needs none |

Open items for the adapter contract:

1. Pin the exact **submit-turn** request (method, body, streaming behaviour) — candidates
   `/api/internal/ask-bot`, `/api/internal/threads`; confirm against `server/index.js` source.
2. Pin the approval decision request that pairs with `/api/decisions`.
3. Confirm SSE event names/types in `/api/events` and map them to protocol v1 phases.
4. Decide auth handling for the packaged app (`/api/auth/session` cookie?) vs headless single-user.

## 6. pi as an engine — how OpenMausBot runs it

OpenMausBot ships a native **`pi` driver** (`resources/server/server/drivers/pi.js`). Verified
from the shipped bundle (v0.1.89):

```js
const PI_ARGS = ["--mode", "rpc", "--no-session"];
const PI_MODEL_UPDATE_ARGS = ["update", "--models", "--no-approve"];
```

- pi is spawned in **RPC mode with `--no-session`** → turns are ephemeral and **never create
  files under `~/.pi/agent/sessions/`**. Interactive pi history stays clean and separate.
- Conversations live in OpenMausBot's own store: `~/.openmausbot/` (`bots.json`, `events/`,
  `memory-journal/`, `attachments/`, per-bot dirs). pi holds no memory of a turn once the RPC
  process/session ends; OpenMausBot replays context per bot/thread.
- Model detection: reads `~/.pi/agent/settings.json` (pi's default model) and uses
  `~/.pi/agent/auth.json` (BYOK credentials, no re-login), probes the catalog through RPC
  `get_available_models`, and refreshes it with `pi update --models --no-approve`.
- Local hosts (Ollama / LM Studio / oMLX / EXO / Unsloth) are **upserted into
  `~/.pi/agent/models.json`** so pi can reach them. Plain cloud usage leaves that file alone.
- Tools/approvals: OpenMausBot injects `pi-mcp-extension.ts` (a JSON-RPC 2.0 stdio MCP
  server) as a configured MCP server; pi's `extension_ui_request` becomes a normal approval
  card in OpenMausBot chat.
- Nothing is written into `~/.pi/agent/extensions/`.

Verified on the machine after a test turn: no new files in `~/.pi/agent/sessions/`, unchanged
mtimes on `models.json` / `settings.json` / `auth.json`, extensions dir untouched (only
`github-token.ts`).

Design consequence for the island: the same RPC path is the lightweight alternative to a
full app — and if we want island chats to be inspectable in pi later, spawn
`pi --mode rpc --session-id <our-id>` instead of `--no-session`.

## 7. Logs and dirs

| Path | Contents |
|---|---|
| `~/.config/openmausbot/` | Electron profile (Chromium caches, `logs/server.log`, `last-run-version.json`) |
| `~/.config/openmausbot/logs/server.log` | harness startup lines |
| `~/.openmausbot/` | transcripts/keys/events per README (created on first use) |
| `~/Applications/OpenMausBot-0.1.89/` | extracted AppImage (can be deleted to uninstall) |
| `~/.local/share/applications/openmausbot.desktop` | launcher (with `--ozone-platform=x11`) |

## 8. Reproduce the checks

```bash
# harness + SSE (app running)
curl -s http://127.0.0.1:8799/api/health
curl -sD- -m 3 -o /dev/null http://127.0.0.1:8799/api/events | head -8
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8799/api/bots

# route strings from the shipped server bundle
grep -ohP '"/api/[a-zA-Z0-9/_:.-]*' \
  ~/Applications/OpenMausBot-0.1.89/app/resources/server/index.js | sort -u

# tray registration
busctl --user list --no-pager | grep -i notifier

# hide the window while keeping the app alive (Hyprland)
hyprctl dispatch movetoworkspacesilent special:oms,class:com.openmausbot.app
```
