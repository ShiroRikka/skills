# MuMu Player 12 Emulator Commands

Complete command reference for `MuMuManager.exe` (MuMu Nebula).

## Prerequisites

- MuMu Nebula installed (`C:\Program Files\Netease\MuMu\nx_main\MuMuManager.exe`)
- The VMs live under `C:\Program Files\Netease\MuMu\vms\MuMuPlayer-12.0-*`

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
| `player_state` | Boot phase: `starting_rom`, `start_finished`, etc. |
| `disk_size_bytes` | VDI disk usage |
| `vt_enabled` / `hyperv_enabled` | Virtualization status |
| `is_main` | Whether this is the primary emulator |

See [info-v-all-fields.md](info-v-all-fields.md) for the complete field reference.

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

# Launch emulator 2 and auto-start Genshin Impact
MuMuManager.exe control -v 2 launch -pkg com.miHoYo.Yuanshen
```

**Success response:** `{"errcode": 0, "errmsg": ""}`

### 3. Shutdown Emulator

```bash
MuMuManager.exe control -v <index> shutdown
```

### 4. Start App Inside Emulator

After emulator is running, launch an app via ADB monkey:

```bash
MuMuManager.exe adb -v <index> -c "shell monkey -p <package_name> -c android.intent.category.LAUNCHER 1"
```

### 5. ADB Convenience Commands

```bash
MuMuManager.exe adb -v <index> -c <command>
```

**Common ADB commands:**

| Command | Purpose |
|---------|---------|
| `connect` | Connect ADB |
| `disconnect` | Disconnect ADB |
| `shell <cmd>` | Run arbitrary shell command |
| `shell pm list packages -3` | List third-party apps |
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
MuMuManager.exe adb -v 2 -c "shell pm list packages -3"
MuMuManager.exe adb -v 2 -c input_text 哈哈
```

### 6. List Installed Apps

```bash
MuMuManager.exe control -v <index> app info --installed
```
Note: Returns `errcode: -201` if the emulator isn't running.

### 7. Close App Inside Emulator

```bash
MuMuManager.exe control -v <index> app close -pkg <package_name>
```

## Common Game Package Names

| Game | Package |
|------|---------|
| 原神 (Genshin Impact) | `com.miHoYo.Yuanshen` |
| 星穹铁道 (Honkai: Star Rail) | `com.miHoYo.hkrpg` |
| 崩坏3 (Honkai Impact 3rd) | `com.miHoYo.htk` |
| 蔚蓝档案 (Blue Archive) | `com.nexon.bluearchive` (varies by region) |

## Full Workflow Example

```bash
# 1. Launch the emulator
"/c/Program Files/Netease/MuMu/nx_main/MuMuManager.exe" control -v 1 launch

# 2. Wait for boot (check is_android_started: true and adb_port present)
"/c/Program Files/Netease/MuMu/nx_main/MuMuManager.exe" info -v all

# 3. Find the app package
"/c/Program Files/Netease/MuMu/nx_main/MuMuManager.exe" adb -v 1 -c "shell pm list packages -3"

# 4. Launch the app
"/c/Program Files/Netease/MuMu/nx_main/MuMuManager.exe" adb -v 1 -c "shell monkey -p com.miHoYo.hkrpg -c android.intent.category.LAUNCHER 1"
```

## Pitfalls

- **Run from bash (git-bash/MSYS):** MuMuManager.exe runs fine from git-bash with proper quoting. Use double quotes around the full path or use `/c/` MSYS paths.
- **ADB command quoting:** The `-c` value containing spaces must be quoted as one argument: `-c "shell pm list packages -3"`, not split across shell tokens.
- **Wait for boot:** After `launch`, do NOT immediately use ADB. Poll `info -v all` until `is_android_started: true` and `adb_port` appears (typically 10-30 seconds).
- **Package names are case-sensitive:** Use exact package name from `pm list packages`.
- **Always use `control shutdown`:** Closing the emulator's main window may not kill the process. Always use the command for clean shutdown.
- **`launch -pkg` vs ADB monkey:** The `launch -pkg` flag starts the app during boot. If the emulator is already running, use `adb shell monkey` instead.