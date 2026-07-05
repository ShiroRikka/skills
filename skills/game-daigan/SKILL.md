---
name: game-daigan
description: Automate daily game farming (代肝/挂机) on MuMu Android emulator using MAA (明日方舟/Arknights) and SRC (星穹铁道/Honkai Star Rail). Use when user wants to start, monitor, or stop emulator-based auto-farming, mentions 代肝/挂机/每日/日常/周常/daily tasks, or wants to automate game tasks on emulator.
compatibility: Windows only. Requires MuMu Player 12 installed, MAA (for Arknights) and/or SRC/StarRailCopilot (for Star Rail) installed via Scoop, and the `mumu-control` Agent Skill available.
metadata:
  version: 0.2.0
  author: Hermes
---

# Game Daigan (游戏自动代肝)

Automate daily game farming tasks on MuMu Android emulator using **MAA** (明日方舟/Arknights) and **SRC** (星穹铁道/Honkai Star Rail) companion programs. Covers the full daigan lifecycle: identifying emulator instances → launching via mumu-control → starting daigan programs → triggering automation → monitoring for completion → clean teardown.

> **This skill handles the game-automation (daigan) layer.** For raw MuMu emulator operations (launch, shutdown, ADB, UI automation), it delegates to the `mumu-control` skill — see [Relationship with mumu-control](#relationship-with-mumu-control).

Does NOT cover: installing the emulator or daigan programs, configuring in-game task settings inside the daigan UI, or general MuMu Player 12 management.

## When to Use

- "帮我代肝明日方舟" / "帮我代肝星铁"
- "启动模拟器并开始挂机"
- "run the daily auto-farm for Arknights / Star Rail"
- "帮我把每日/周常做了"
- User wants to start, monitor, or stop emulator-based game automation
- Any request involving 代肝, 挂机, 每日, daily farming, MAA, SRC, or auto-farming on MuMu

## Prerequisites

- **MuMu Player 12** installed (path auto-detected by `mumu-control`)
- **MAA** (for Arknights) via Scoop: `~/scoop/apps/maa/current/MAA.exe`
- **SRC / StarRailCopilot** (for Star Rail) via Scoop: `~/scoop/apps/StarRailCopilot/current/src.exe`
- **`mumu-control` skill** available in the same skills directory (`skills/mumu-control/SKILL.md`)

## Relationship with mumu-control

The official [`mumu-control`](../mumu-control/SKILL.md) skill covers all MuMu Player 12 operations:

| Operation | Handled by |
|-----------|-----------|
| Find installation path | `mumu-control` — auto-detection chain |
| List emulator instances (`info`) | `mumu-control` — `mumu-cli.exe info --vmindex all` |
| Launch/shutdown emulator | `mumu-control` — `control launch` / `control shutdown` |
| Wait for boot | `mumu-control` — poll `player_state == start_finished` |
| ADB connection & commands | `mumu-control` — bundled `adb.exe`, app management |
| UI automation (tap, swipe, OCR) | `mumu-control` — Python scripts (`tap.py`, `uiscan.py`, etc.) |

**This skill (`game-daigan`) only adds game-specific knowledge** — which emulator instance maps to which game, how to start MAA/SRC with correct flags and environment, how to verify they're running, and how to detect task completion.

When following the procedure below, load the `mumu-control` skill for emulator-related steps and consult its reference documentation there. Do not re-document MuMu CLI commands.

## Workflow Overview

```
1. Identify emulators     → mumu-cli info --vmindex all (mumu-control)
2. Launch emulator(s)     → mumu-cli control launch (mumu-control)
3. Wait for ready         → poll info until is_android_started (mumu-control)
4. Start daigan program   → MAA.exe / src.exe --run (game-daigan)
5. Verify daigan window   → confirm UI visible (game-daigan)
6. Start automation       → click button if needed (game-daigan)
7. Inform user            → "已经开始代肝了，完成后叫我"
8. (user signals done)    → kill daigan processes (game-daigan)
9. Shutdown emulator(s)   → mumu-cli control shutdown (mumu-control)
```

**Default:** if user doesn't specify a game, run **both** MAA and SRC in parallel. If one is specified, run only that one.

## Procedure

### Game Selection

Parse the user's request:

| Mention | Game |
|---------|------|
| "明日方舟" / "Arknights" / "MAA" | → MAA |
| "星穹铁道" / "星铁" / "Star Rail" / "SRC" | → SRC |
| 未指定 / "都肝" / "全部" / "both" | → **Run both** |

Keep a running list of target games for cleanup steps. Store the background process session IDs from Step 4.

---

### 1. Identify Emulators

Consult the `mumu-control` skill (read `skills/mumu-control/SKILL.md`) for the full `mumu-cli.exe` command reference. Run:

```bash
mumu-cli.exe info --vmindex all
```

Parse the JSON. Identify the `--vmindex` for each target game by matching `"name"`:

| Game | Expected name |
|------|---------------|
| 明日方舟 | "明日方舟" |
| 星穹铁道 | "星穹铁道" |

See [current-setup.md](references/mumu-emulator/current-setup.md) for this machine's emulator layout.

---

### 2. Launch Emulator(s)

Using `mumu-control`:

```bash
# Launch each target emulator
mumu-cli.exe control --vmindex <INDEX_1> launch
mumu-cli.exe control --vmindex <INDEX_2> launch
```

---

### 3. Wait for Ready

Poll `mumu-cli.exe info --vmindex all` in a loop until **all** conditions hold for each target index:

- `"is_process_started": true`
- `"is_android_started": true`
- `"player_state": "start_finished"`
- `"adb_port"` field appears

Typical wait: 30–120 seconds per emulator. Both boot independently — poll the full list each cycle.

---

### 4. Launch Daigan Program(s)

Launch daigan programs in **parallel** via `terminal(background=true)`. Save session IDs.

**MAA (if active):**

```bash
"MAA.exe"
```

Path: `~/scoop/apps/maa/current/MAA.exe`

Launch directly — the window opens and you proceed to Step 6 to click "Link Start!".

**SRC (if active):**

```bash
cd "~/scoop/apps/StarRailCopilot/current" && env -u PYTHONPATH ./src.exe --run src
```

> ⚠️ SRC requires a clean environment: always `cd` into its install directory and clear `PYTHONPATH` before launching. Running from elsewhere or with a contaminated `PYTHONPATH` causes import errors.

The `--run src` flag auto-starts automation (see [src-cli.md](references/src-cli.md)). The config name (`src`) matches the default instance — verify by checking `config/deploy.yaml` if unsure.

---

### 5. Verify Daigan Window(s)

Wait 5–15 seconds after launch, then confirm each program's UI is visible using `computer_use`.

**MAA:** `computer_use(action='capture', app="MAA")` — verify main UI tabs (一键长草, 自动战斗, 小工具, 设置) and the "Link Start!" button.

**SRC (with `--run`):** `computer_use(action='capture', app="src")` — verify scheduler UI. Task starts immediately — button may show "停止".

**SRC (without `--run`, fallback):** `computer_use(action='capture', app="src")` — button should show "启动".

---

### 6. Start Daigan(s)

**MAA (if active):**
1. `computer_use(action='capture', mode='som', app="MAA")` — get numbered overlays
2. Click the "Link Start!" button by element index
3. Re-capture and confirm button text changed to "停止"

**SRC (if active, without `--run`):**
1. `computer_use(action='capture', mode='som', app="src")` — get numbered overlays
2. Click the "启动" button by element index
3. Re-capture and confirm button text changed to "停止"

**SRC (if active, with `--run`):** Skip this step — automation started on launch.

---

### 7. Inform User

All daigan programs are running. Tell the user:

> **"已经开始了！肝完了叫我来收尾就好~"**

Stop here. Wait for the user to return and say something like "肝完了", "结束了", "收尾", "关了吧", "done".

---

### 8. Close Daigan Program(s)

Use the session IDs saved from Step 4:

```
process(action='kill', session_id='<maa_session>')
process(action='kill', session_id='<src_session>')
```

No need to click "停止" inside the GUI — just kill directly.

---

### 9. Shutdown Emulator(s)

Using `mumu-control`:

```bash
mumu-cli.exe control --vmindex <INDEX_1> shutdown
mumu-cli.exe control --vmindex <INDEX_2> shutdown
```

Verify: `mumu-cli.exe info --vmindex all` — confirm each target shows `"is_process_started": false`.

## Completion Detection (Optional)

You can monitor task progress without relying on the user:

**MAA** — check `~/scoop/apps/maa/current/debug/gui.log` for the keyword `任务已全部完成`. See [maa-log.md](references/maa-log.md) for details.

**SRC** — check `~/scoop/apps/StarRailCopilot/current/log/{YYYY-MM-DD}_{config_name}.txt` for `No task pending`. See [src-cli.md](references/src-cli.md#log-based-completion-detection) for details.

Poll logs every 30–60 seconds when monitoring.

## Pitfalls

- **SRC requires a clean environment**: Always `cd` into its install directory and clear `PYTHONPATH` before launching. Running from elsewhere or with a contaminated `PYTHONPATH` causes import errors. This applies even when using `--run`.
- **`--run` with wrong config name**: SRC starts but does nothing. Config name is case-sensitive and must match an existing instance name under `config/`. Check `config/deploy.yaml` if unsure.
- **MAA window title is unstable**: It includes version/build info that updates with each release. Never match on window title — use `app="MAA"` for process-based targeting.
- **Background launch ≠ instant ready**: Daigan programs take 5–30 seconds to render their first window. Use a delay or poll logs before capturing.
- **Multi-instance ambiguity**: If multiple emulators could match the game name, check `"is_main"` or ask the user.
- **Locale-dependent button text**: The skill assumes Chinese UI (MAA/SRC default). If the locale is different, button strings will differ.
- **SRC `--run` auto-starts but does NOT auto-stop**: Still need to kill the process (Step 8) when done.

## Verification

After cleanup:

1. `mumu-cli.exe info --vmindex all` — confirm each target emulator shows `"is_process_started": false`
2. `process(action='list')` — confirm no daigan processes remain