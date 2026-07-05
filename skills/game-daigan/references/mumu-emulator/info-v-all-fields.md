# MuMuManager.exe info -v all — Complete Field Reference

Source: MuMu Player 12 official documentation.

## Common Fields (always present)

```json
{
  "adb_host_ip": "127.0.0.1",       // ADB host IP (only when running)
  "adb_port": 16384,                 // ADB port (only when running)
  "created_timestamp": 1721180910349678, // Emulator creation timestamp
  "disk_size_bytes": 284192948,      // VDI disk usage in bytes
  "error_code": 0,                   // Emulator list error code
  "headless_pid": 20868,             // VM process PID (only when running)
  "hyperv_enabled": false,           // Hyper-V virtualization enabled
  "index": "0",                      // Emulator index
  "is_android_started": false,       // Android OS booted successfully
  "is_main": true,                   // Is the primary emulator
  "is_process_started": true,        // Process is running
  "launch_err_code": 0,              // Launch error code (only when running)
  "launch_err_msg": "",              // Launch error description (only when running)
  "launch_time": 2222,               // Emulator runtime in sec (only when running)
  "main_wnd": "00840F4E",            // Main window handle (only when running)
  "name": "MuMu模拟器12",               // Emulator display name
  "pid": 15112,                      // Shell process PID (only when running)
  "player_state": "starting_rom",    // Shell boot phase (only when running)
  "render_wnd": "00B30C7A",          // Render window handle (only when running)
  "vt_enabled": true                 // VT-x virtualization enabled (only when running)
}
```

## Field Behavior

| Field | Non-Running | Running |
|-------|:-----------:|:-------:|
| `adb_host_ip` | absent | present |
| `adb_port` | absent | present |
| `headless_pid` | absent | present |
| `launch_err_code` | absent | present |
| `launch_err_msg` | absent | present |
| `launch_time` | absent | present |
| `main_wnd` | absent | present |
| `pid` | absent | present |
| `player_state` | absent | present |
| `render_wnd` | absent | present |
| `vt_enabled` | absent | present |

## Player State Values

| Value | Meaning |
|-------|---------|
| `starting_rom` | VM booting |
| `start_finished` | Boot complete, ready |

## Error Codes

| Code | Meaning |
|:----:|---------|
| 0 | Success / No error |
| non-zero | See `launch_err_msg` for description |