---
name: game-daigan
description: Automate daily game farming on MuMu Android emulator with MAA (Arknights/明日方舟) and SRC (Star Rail/星穹铁道). Use when user wants to start/stop emulator-based auto-farming, or mentions 代肝, 挂机, 每日, daily tasks, or emulator automation.
metadata:
  version: 0.1.0
  author: Hermes
  platforms:
    - windows
  hermes:
    tags: [Gaming, Automation, MuMu]
---

# Game Daigan (游戏自动代肝)

Automate daily game farming tasks on MuMu Android emulator using MAA (明日方舟/Arknights) and SRC (星穹铁道/Honkai Star Rail) companion programs. Covers the full lifecycle: identifying emulator instances, launching/shutdown via MuMuManager, starting the daigan programs, triggering automation, monitoring for completion, and clean teardown. Does NOT cover installing the emulator or daigan programs, nor configuring in-game task settings inside the daigan UI.

## When to Use

- "帮我代肝明日方舟" / "帮我代肝星铁"
- "启动模拟器并开始挂机"
- "run the daily auto-farm for Arknights / Star Rail"
- User wants to start, monitor, or stop emulator-based game automation
- "帮我把每日/周常做了"

## Prerequisites

- **MuMu Nebula** installed under `C:\Program Files\Netease\MuMu\nx_main\`
- **MAA** (for Arknights) installed via Scoop: `~/scoop/apps/maa/current/MAA.exe`
- **SRC / StarRailCopilot** (for Star Rail) via Scoop: `~/scoop/apps/StarRailCopilot/current/src.exe`
- **cua-driver** installed (for `computer_use` actions)
- Emulator instances known — first run `MuMuManager.exe info -v all` to see the map

## How to Run

Invoke through `terminal` and `computer_use` tools following the 10-step procedure below. The canonical flow:

1. `terminal` — `info -v all` to identify target emulator by name
2. `terminal` — launch emulator via `MuMuManager.exe control -v <N> launch`
3. `terminal` / `process` — poll until emulator fully started
4. `terminal(background=true)` — launch daigan program
5. `computer_use` — wait for daigan window to appear and stabilize
6. `computer_use` — click start button, verify button text changes
7. `computer_use(mode=vision)` — periodically inspect for completion signal
8. `computer_use` — click stop button
9. `process(action=kill)` — close daigan program
10. `terminal` — shutdown emulator

## Quick Reference

See [emulator-commands.md](references/mumu-emulator/emulator-commands.md) for the full MuMuManager command reference (launch, shutdown, ADB, app management, package names). For commands not covered there (create, clone, setting, etc.), consult the [official MuMuManager docs](https://mumu.163.com/help/20240807/40912_1170006.html).

### Quick Command Summary

| Action | Command |
|---|---|
| List all emulators | `MuMuManager.exe info -v all` |
| Launch emulator | `MuMuManager.exe control -v <N> launch` |
| Shutdown emulator | `MuMuManager.exe control -v <N> shutdown` |
| ADB command | `MuMuManager.exe adb -v <N> -c <cmd>` |

### Daigan Program Paths (via Scoop `current`)

| Game | Program | Path | Launch Note |
|---|---|---|---|
| 明日方舟 | MAA | `~/scoop/apps/maa/current/MAA.exe` | Direct launch |
| 星穹铁道 | SRC | `~/scoop/apps/StarRailCopilot/current/src.exe` | `cd` to dir + `env -u PYTHONPATH` |

> **SRC tip:** Add `--run src` to auto-start automation on launch, skipping the button-click step. See [src-cli.md](references/src-cli.md) for details.

### Computer Use Window Targets

| Program | `app=` parameter | Reason |
|---|---|---|
| MAA | `app="MAA"` | Window title changes daily, use process match |
| SRC | `app="src"` | Window title is stable |

### Button Semantics

| Program | Start | Running indicator | Auto-start |
|---|---|---|---|
| MAA | "Link Start!" | Button becomes "停止" | No (must click) |
| SRC | "启动" | Button becomes "停止" | Yes — `--run src` skips click, see [src-cli.md](references/src-cli.md) |

### Info JSON Key Fields

| Field | Meaning |
|---|---|
| `name` | Emulator display name (matches the game) |
| `index` | Numeric index for `-v` flag |
| `is_process_started` | VM process running |
| `is_android_started` | Android OS fully booted |
| `player_state` | `"start_finished"` = ready |

See [info-v-all-fields.md](references/mumu-emulator/info-v-all-fields.md) for the complete field reference.

## Procedure

### 1. Identify the Target Emulator

Run through `terminal`:

```
"C:\Program Files\Netease\MuMu\nx_main\MuMuManager.exe" info -v all
```

Parse the JSON. Match `"name"` to the game (e.g. "明日方舟" → index 0, "星穹铁道" → index 1). Record the `index`. See [info-v-all-fields.md](references/mumu-emulator/info-v-all-fields.md) for the complete field reference, or [current-setup.md](references/mumu-emulator/current-setup.md) for this machine's emulator layout.

### 2. Launch the Emulator

```
MuMuManager.exe control -v <INDEX> launch
```

### 3. Wait for Emulator Ready

Poll `info -v all` in a loop until **all** of:
- `"is_process_started": true`
- `"is_android_started": true`
- `"player_state": "start_finished"`
- `"adb_port"` field appears

Typical wait: 30–120 seconds.

### 4. Launch the Daigan Program

Use `terminal(background=true, notify_on_complete=true)`.

**MAA:**
```bash
"C:\Users\shiro\scoop\apps\maa\current\MAA.exe"
```

**SRC (preferred — auto-start with `--run`):**
```bash
cd "C:/Users/shiro/scoop/apps/StarRailCopilot/current" && env -u PYTHONPATH ./src.exe --run src
```

**SRC (manual — without `--run`, fallback):**
```bash
cd "C:/Users/shiro/scoop/apps/StarRailCopilot/current" && env -u PYTHONPATH ./src.exe
```

### 5. Wait for Daigan Window

Use `computer_use(action='capture', app="MAA"|"src")` to confirm the UI is visible. Do not capture immediately — wait 5–15 seconds after launch for the window to render.

**MAA:** Verify main UI tabs (一键长草, 自动战斗, 小工具, 设置) and the "Link Start!" button are present.

**SRC (with `--run`):** Verify the scheduler UI (左侧导航, 日志区域). The task starts immediately — the start button may already show "停止". Confirm task progress appears in the log area.

**SRC (without `--run`):** Verify the scheduler UI (左侧导航, 启动按钮, 日志区域) is present with the button showing "启动".

### 6. Start Daigan

**MAA / SRC (without `--run`):**
1. `computer_use(action='capture', mode='som', app="XXX")` — get numbered overlays
2. Click the start button by element index
3. Re-capture and confirm button text changed to "停止"

If the SOM index doesn't capture the button cleanly, fall back to `mode='vision'` with coordinates.

**SRC (with `--run`):** Skip this step — automation started automatically on launch. Proceed directly to monitoring.

### 7. Monitor for Completion

**SRC (preferred — log-based):**

```bash
tail -n 100 ~/scoop/apps/StarRailCopilot/current/log/$(date +%Y-%m-%d)_src.txt
```

Look for:
```
No task pending
Wait until ... for task `Restart`
```

See [src-cli.md](references/src-cli.md#log-based-completion-detection) for details. If the log file doesn't exist yet, wait and retry.

**MAA (preferred — log-based):**

```bash
tail -n 30 ~/scoop/apps/maa/current/debug/gui.log
```

Look for:
```
任务已全部完成！
(用时 0h XXm XXs)
理智将在 ... 回满。
```

See [maa-log.md](references/maa-log.md) for details, pitfalls, and fallback.

**Vision-based fallback (both):** If the log file is inaccessible or uncertain, use `computer_use(action='capture', mode='vision', app="MAA"|"src")` every 2–5 minutes:
- **MAA:** "Link Start!" / "停止" button state, task progress messages in logs
- **SRC:** Queue showing "无任务", scheduler idle state, button reverted to "启动"

### 8. Stop Daigan

When completion detected:
1. `computer_use(action='capture', mode='som', app="XXX")`
2. Click the "停止" button
3. Verify it reverts to the original start text

### 9. Close Daigan Program

```
process(action='kill', session_id='<session_id>')
```

The session_id is from the `terminal(background=true)` output in Step 4.

### 10. Shutdown Emulator

```
MuMuManager.exe control -v <INDEX> shutdown
```

Verify with `info -v all` that `"is_process_started": false` for the target index.

## Pitfalls

- **SRC requires a clean environment**: Always `cd` into its install directory and clear `PYTHONPATH` before launching. Running from elsewhere or with a contaminated `PYTHONPATH` causes import errors. This applies even when using `--run`.
- **`--run` ≠ no monitor**: Using `--run src` auto-starts SRC tasks but does NOT auto-stop. Still need to monitor completion and click "停止" (Step 8).
- **`--run` with wrong config name**: SRC starts but does nothing. Config name is case-sensitive and must match an existing instance name under `config/`.
- **MAA window title is unstable**: It includes version/build info that updates with each release. Never match on window title — use `app="MAA"` for process-based targeting.
- **Background launch ≠ instant ready**: Daigan programs take 5–30 seconds to render their first window. Use `notify_on_complete` or poll logs before capturing.
- **-201 errors from MuMuManager**: Commands like `app info --installed` return `errcode: -201` if the target emulator isn't running. Always verify emulator state first.
- **Multi-instance ambiguity**: If multiple emulators could match the game name, check `"is_main"` or ask the user.
- **Locale-dependent button text**: The skill assumes Chinese UI (MAA/SRC default). If the locale is different, button strings will differ and need updating.

## Verification

After completing the full workflow:

1. `MuMuManager.exe info -v all` — confirm target emulator shows `"is_process_started": false`
2. `process(action='list')` — confirm no daigan processes remain
3. `computer_use(action='capture', app="MAA"|"src")` — confirm daigan window is gone