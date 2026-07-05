---
name: mumu-emulator
description: Manage MuMu Player 12 Android emulator instances on Windows — launch, shutdown, ADB commands, query status and find installed app packages.
tags: [mumu, emulator, android, adb, windows, mumuplayer]
category: computer-use
---

# MuMu Player 12 Emulator Management (Windows)

Manage MuMu Player 12 Android emulator instances on Windows using `MuMuManager.exe`.

## Prerequisites

- MuMu Player 12 installed (`C:\Program Files\Netease\MuMu\nx_main\MuMuManager.exe`)
- The VMs live under `C:\Program Files\Netease\MuMu\vms\MuMuPlayer-12.0-*`

## Common Path

```
C:\Program Files\Netease\MuMu\nx_main\MuMuManager.exe
```

Create a shell alias for convenience:

```bash
alias mumu="/c/Program\ Files/Netease/MuMu/nx_main/MuMuManager.exe"
```

## Commands

### 1. Query All Emulator Info

```bash
MuMuManager.exe info -v all
```

**Key output fields:**

| Field | Meaning |
|-------|---------|
| `index` | Emulator index (used for `-v` flag) |
| `name` | User-friendly emulator name |
| `is_android_started` | Android OS fully booted |
| `is_process_started` | Emulator process is running |
| `adb_host_ip` / `adb_port` | ADB connection (only when running) |
| `pid` / `headless_pid` | Process PIDs (only when running) |
| `player_state` | Boot phase (only when running): `starting_rom`, `start_finished`, etc. |
| `disk_size_bytes` | VDI disk usage |
| `vt_enabled` / `hyperv_enabled` | Virtualization status |
| `is_main` | Whether this is the primary emulator |

**Running instances** include extra fields: `adb_host_ip`, `adb_port`, `pid`, `headless_pid`, `main_wnd`, `render_wnd`, `launch_err_code`, `launch_err_msg`, `launch_time`, `player_state`, `vt_enabled`.

### 2. Launch Emulator

```bash
MuMuManager.exe control -v <index> launch [-pkg <package_name>]
```

- `-v <index>` — emulator index (from `info -v all`)
- `-pkg <package_name>` — optional: auto-start an app after boot

Examples:
```bash
# Launch emulator 0
MuMuManager.exe control -v 0 launch

# Launch emulator 2 and auto-start 原神
MuMuManager.exe control -v 2 launch -pkg com.miHoYo.Yuanshen
```

**Success response:** `{"errcode": 0, "errmsg": ""}`

### 3. Shutdown Emulator

```bash
MuMuManager.exe control -v <index> shutdown
```

### 4. ADB Convenience Commands

```bash
MuMuManager.exe adb -v <index> -c <command>
```

**Common ADB commands:**

| Command | Purpose |
|---------|---------|
| `connect` | Connect ADB |
| `disconnect` | Disconnect ADB |
| `shell <cmd>` | Run arbitrary shell command |
| `input_text <text>` | Type text input |
| `go_back` | Android back button |
| `go_home` | Android home button |
| `go_task` | Android recent tasks |
| `volume_up` | Volume up |
| `volume_down` | Volume down |
| `volume_mute` | Mute |

Examples:
```bash
MuMuManager.exe adb -v 2 -c connect
MuMuManager.exe adb -v 2 -c "shell getprop ro.opengles.version"
MuMuManager.exe adb -v 2 -c "shell pm list packages -3"    # List third-party apps
MuMuManager.exe adb -v 2 -c input_text 哈哈
```

## How to Find and Launch an App Inside the Emulator

### Step 1: List third-party installed apps

```bash
MuMuManager.exe adb -v <index> -c "shell pm list packages -3"
```

Output format: `package:com.miHoYo.hkrpg`

Common game package names:
| Game | Package |
|------|---------|
| 原神 (Genshin Impact) | `com.miHoYo.Yuanshen` |
| 星穹铁道 (Honkai: Star Rail) | `com.miHoYo.hkrpg` |
| 崩坏3 (Honkai Impact 3rd) | `com.miHoYo.htk` |
| 蔚蓝档案 (Blue Archive) | `com.nexon.bluearchive` (varies by region) |

### Step 2: Launch the app via ADB monkey

```bash
MuMuManager.exe adb -v <index> -c "shell monkey -p <package_name> -c android.intent.category.LAUNCHER 1"
```

## Full Workflow Example (Launch Emulator → Find App → Start App)

```bash
# 1. Launch the emulator (background task)
"/c/Program Files/Netease/MuMu/nx_main/MuMuManager.exe" control -v 1 launch

# 2. Wait for boot (check `is_android_started: true` and `adb_port` present)
"/c/Program Files/Netease/MuMu/nx_main/MuMuManager.exe" info -v all

# 3. Find the app package
"/c/Program Files/Netease/MuMu/nx_main/MuMuManager.exe" adb -v 1 -c "shell pm list packages -3"

# 4. Launch the app
"/c/Program Files/Netease/MuMu/nx_main/MuMuManager.exe" adb -v 1 -c "shell monkey -p com.miHoYo.hkrpg -c android.intent.category.LAUNCHER 1"
```

## Pitfalls

- **Run from bash (git-bash/MSYS):** On Windows, MuMuManager.exe runs fine from git-bash with proper quoting. Use double quotes around the full path and escape spaces or use `/c/` MSYS paths.
- **ADB command quoting:** The `-c` value containing spaces must be quoted as one argument: `-c "shell pm list packages -3"`, not split across shell tokens.
- **Wait for boot:** After `launch`, do NOT immediately use ADB. Poll `info -v all` until `is_android_started: true` and `adb_port` appears (typically 10-30 seconds). Using ADB too early returns connection errors.
- **Package names are case-sensitive:** Use exact package name from `pm list packages`.
- **Simulator may need to stay open:** Closing the emulator's main window may or may not kill the process depending on settings. Always use `control -v <index> shutdown` for clean shutdown.
- **Using `launch -pkg` vs ADB monkey:** The `launch -pkg` flag starts the app during boot. If the emulator is already running, use `adb shell monkey` instead.