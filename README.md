# claude-buddy-bridge

Visualize your Claude Code tool calls on the **Claude Buddy** desktop-pet hardware over BLE.

This repository is the **plugin** — a thin hook layer that registers Claude Code hooks and
forwards each event to a local daemon. It is intentionally small (no firmware, no BLE stack,
no binaries) so that `claude plugin marketplace add` stays fast.

```
Claude Code hook → hook_bridge.py → TCP 127.0.0.1:57320 → claude-buddy daemon → BLE → ESP32
```

---

## ⚠️ The plugin alone is not enough

Installing this plugin only registers the hooks. To actually see anything on hardware you also need:

1. **The daemon runtime** installed separately at `~/.claude-buddy/` (the BLE daemon + pairing tool).
2. **A paired Claude Buddy device** (ESP32 clock or panel).

The daemon runtime, firmware, and the one-click setup tool live in the main project:
**https://github.com/FreakStudioCN/MicroPython_Claude_Assistant**

If the daemon is not running, every hook **fails open** — Claude Code is never blocked or slowed,
the hardware simply shows nothing.

---

## Install

```bash
claude plugin marketplace add ChrisWu132/claude-buddy-plugin
claude plugin install claude-buddy-bridge@claude-buddy
```

**Requirement:** Python 3 must be on `PATH`. The hook command invokes `pythonw` / `python`
to run `daemon/hook_bridge.py`.

---

## Verify your install

**1. The plugin is registered and enabled**

```bash
claude plugin list
```

You should see `claude-buddy-bridge@claude-buddy` with `Status: ✔ enabled`.

**2. The bridge runs — no daemon or hardware needed**

The bridge is *fail-open*: given any hook event it always replies `{}` and never blocks Claude
Code, even when no daemon is running. Clone this repo and feed it a sample event:

```bash
# macOS / Linux
echo '{}' | python daemon/hook_bridge.py
echo '{"hook_event_name":"PreToolUse","tool_name":"Read","tool_input":{"file_path":"x"}}' | python daemon/hook_bridge.py
```

```powershell
# Windows PowerShell
'{}' | python daemon\hook_bridge.py
```

Each command should print `{}`. That means the forwarder loads and runs.

**3. Validate the manifest (for contributors / before publishing)**

```bash
claude plugin validate .
```

Expected: `✔ Validation passed` (a couple of cosmetic warnings about description/author are fine).

**4. Full hardware test — needs the daemon + a paired device**

Install and start the daemon runtime from the main project
([FreakStudioCN/MicroPython_Claude_Assistant](https://github.com/FreakStudioCN/MicroPython_Claude_Assistant)),
then run any command in a Claude Code session — the device should change state as tools run.
The main project ships a `claude-buddy-smoke` tool to confirm the daemon link **without** hardware.

> Anonymous reachability (can a stranger install it?) is guaranteed by the repo being **public** —
> verify with `git clone https://github.com/ChrisWu132/claude-buddy-plugin.git` from a machine with
> no GitHub credentials; a successful clone means `claude plugin marketplace add` can fetch it too.

---

## How it works

`hooks/hooks.json` registers 8 Claude Code events:

| Event | Used for |
|---|---|
| `UserPromptSubmit` | turn start (clears idle) |
| `PreToolUse` / `PostToolUse` / `PostToolUseFailure` | tool running / done / error state |
| `Notification` | approval-prompt reminder |
| `Stop` / `StopFailure` | task done / task error |
| `SessionEnd` | session cleanup |

Each event is read from stdin, normalized into a small v2 envelope, and sent to the daemon on
`127.0.0.1:57320`. The bridge always returns `{}` (it never participates in approval). If the
daemon is unreachable, the bridge best-effort spawns it from `~/.claude-buddy/` and fails open
for the current event — the next hook reconnects.

The device is **display-only**: it shows light / voice / screen state. All approvals happen in
the Claude Code terminal UI as usual.

---

## Customization

Edit `daemon/risk_config.py` to tune how operations are classified (`safe` / `normal` / `critical`)
for offline auto-approval behavior. The file is optional — if removed, built-in defaults apply.

---

## License

No open-source license is attached; all rights reserved by FreakStudio.
