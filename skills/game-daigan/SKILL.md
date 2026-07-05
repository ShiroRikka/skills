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

Invoke through `terminal` and `computer_use` tools following the procedure below. The canonical flow:

**Default:** if user doesn't specify a game, run **both** 明日方舟 (MAA) and 星穹铁道 (SRC) in **parallel**. If one is specified, run only that one.

```
1.  terminal → info -v all               identify all emulators
2.  terminal → control launch <index>    launch target emulator(s)
3.  terminal → poll info                  wait for ready (each)
4.  terminal(background) → MAA / src      start daigan program(s)
5.  computer_use → capture                confirm windows
6.  computer_use → click start            MAA only (SRC --run skips)
7.  terminal → tail log                   wait for completion
8.  process(kill)                         close daigan program(s)
9.  terminal → control shutdown           shut down emulator(s)
```

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

The steps below show the **both-games** flow (superset). For single-game mode, simply skip the other game's sub-steps.

### Game Selection

Parse the user's request:
- Mention of "明日方舟" / "Arknights" / "MAA" → run MAA
- Mention of "星穹铁道" / "星铁" / "Star Rail" / "SRC" → run SRC
- **未指定 / "都肝" / "全部" / "both" → run both in parallel**

Keep a running list of target games for cleanup steps.

### 1. Identify Emulators

Run once:

```
"C:\Program Files\Netease\MuMu\nx_main\MuMuManager.exe" info -v all
```

Parse the JSON. Identify the index for each target game by matching `"name"`:

| Game | Expected name | 
|------|---------------|
| 明日方舟 | "明日方舟" |
| 星穹铁道 | "星穹铁道" |

Record each target's index. See [info-v-all-fields.md](references/mumu-emulator/info-v-all-fields.md) for the complete field reference, or [current-setup.md](references/mumu-emulator/current-setup.md) for this machine's emulator layout.

### 2. Launch Emulator(s)

Launch each target emulator. For both games, issue both commands in sequence (they return immediately):

```bash
MuMuManager.exe control -v <INDEX_ARKNIGHTS> launch
MuMuManager.exe control -v <INDEX_STARRAIL> launch
```

### 3. Wait for Ready

Poll `info -v all` in a loop until **all** conditions hold for **each** target index:
- `"is_process_started": true`
- `"is_android_started": true`
- `"player_state": "start_finished"`
- `"adb_port"` field appears

Typical wait: 30–120 seconds per emulator. Both emulators boot independently — poll the full list each cycle.

### 4. Launch Daigan Program(s)

For each target game, start the daigan program in the background. Both can run simultaneously.

**MAA (if active):**
```bash
"MAA.exe"
```

**SRC (if active) — preferred with `--run`:**
```bash
cd "C:/Users/shiro/scoop/apps/StarRailCopilot/current" && env -u PYTHONPATH ./src.exe --run src
```

**SRC (manual — without `--run`, fallback):**
```bash
cd "C:/Users/shiro/scoop/apps/StarRailCopilot/current" && env -u PYTHONPATH ./src.exe
```

### 5. Wait for Daigan Window(s)

For each active program, confirm its UI is visible. Wait 5–15 seconds after launch before capturing.

**MAA (if active):** `computer_use(action='capture', app="MAA")` — verify main UI tabs (一键长草, 自动战斗, 小工具, 设置) and the "Link Start!" button.

**SRC (if active, with `--run`):** `computer_use(action='capture', app="src")` — verify scheduler UI (左侧导航, 日志区域). Task starts immediately — button may show "停止".

**SRC (if active, without `--run`):** `computer_use(action='capture', app="src")` — verify scheduler UI (左侧导航, 启动按钮, 日志区域) with button showing "启动".

### 6. Start Daigan(s)

**MAA (if active) / SRC (if active, without `--run`):**
1. `computer_use(action='capture', mode='som', app="MAA"|"src")` — get numbered overlays
2. Click the start button by element index
3. Re-capture and confirm button text changed to "停止"

If the SOM index doesn't capture the button cleanly, fall back to `mode='vision'` with coordinates.

**SRC (if active, with `--run`):** Skip this step — automation started on launch.

### 7. Monitor for Completion

Monitor each active program independently. Both can be checked in parallel.

**SRC (if active — log-based):**

```bash
tail -n 100 ~/scoop/apps/StarRailCopilot/current/log/$(date +%Y-%m-%d)_src.txt
```

Look for:
```
No task pending
Wait until ... for task `Restart`
```

See [src-cli.md](references/src-cli.md#log-based-completion-detection). If log doesn't exist yet, wait and retry.

**MAA (if active — log-based):**

```bash
tail -n 30 ~/scoop/apps/maa/current/debug/gui.log
```

Look for:
```
任务已全部完成！
(用时 0h XXm XXs)
理智将在 ... 回满。
```

See [maa-log.md](references/maa-log.md).

**Vision-based fallback:** If log files are inaccessible, use `computer_use(action='capture', mode='vision', app="MAA"|"src")` every 2–5 minutes:
- **MAA:** "任务已全部完成" or "任务完成" text in the log/status panel
- **SRC:** "无任务" in the queue area

> ⚠️ When tasks are done, the button stays as "停止". Do not click it — proceed to Step 8 to kill directly.

### 8. Close Daigan Program(s)

Once **all** active programs are confirmed complete, kill each:

```
process(action='kill', session_id='<maa_session>')
process(action='kill', session_id='<src_session>')
```

The session IDs are from the `terminal(background=true)` outputs in Step 4.

### 9. Shutdown Emulator(s)

Shut down each target emulator:

```
MuMuManager.exe control -v <INDEX_ARKNIGHTS> shutdown
MuMuManager.exe control -v <INDEX_STARRAIL> shutdown
```

Verify with `info -v all` that each target emulator shows `"is_process_started": false`.

## Pitfalls

- **SRC requires a clean environment**: Always `cd` into its install directory and clear `PYTHONPATH` before launching. Running from elsewhere or with a contaminated `PYTHONPATH` causes import errors. This applies even when using `--run`.
- **`--run` ≠ no monitor**: Using `--run src` auto-starts SRC tasks but does NOT auto-stop. Still need to monitor completion (Step 7) then kill the process (Step 8).
- **`--run` with wrong config name**: SRC starts but does nothing. Config name is case-sensitive and must match an existing instance name under `config/`.
- **MAA window title is unstable**: It includes version/build info that updates with each release. Never match on window title — use `app="MAA"` for process-based targeting.
- **Background launch ≠ instant ready**: Daigan programs take 5–30 seconds to render their first window. Use `notify_on_complete` or poll logs before capturing.
- **-201 errors from MuMuManager**: Commands like `app info --installed` return `errcode: -201` if the target emulator isn't running. Always verify emulator state first.
- **Multi-instance ambiguity**: If multiple emulators could match the game name, check `"is_main"` or ask the user.
- **Locale-dependent button text**: The skill assumes Chinese UI (MAA/SRC default). If the locale is different, button strings will differ and need updating.

## Verification

After completing the full workflow:

1. `MuMuManager.exe info -v all` — confirm each target emulator shows `"is_process_started": false`
2. `process(action='list')` — confirm no daigan processes remain
3. `computer_use(action='capture', app="MAA"|"src")` — confirm daigan windows are gone