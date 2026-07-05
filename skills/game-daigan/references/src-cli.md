# StarRailCopilot (SRC) CLI & Config Reference

SRC supports auto-running tasks on startup, eliminating the need for manual `computer_use` button clicks.

## Auto-Run on Startup (Two Ways)

### Way 1: CLI `--run` flag (recommended)

Pass the config name when launching:

```bash
cd ~/scoop/apps/StarRailCopilot/current && env -u PYTHONPATH ./src.exe --run src
```

Or via Python:

```bash
cd ~/scoop/apps/StarRailCopilot/current && env -u PYTHONPATH python gui.py --run src
```

Multiple configs: `--run src src2`

### Way 2: `config/deploy.yaml` setting

Edit `config/deploy.yaml` in the SRC install directory:

```yaml
Webui:
  # --run. Auto-run specified config when startup
  # 'null' default no specified config
  # '["src"]' specified "src" config
  # '["src","src2"]' specified "src" "src2" configs
  Run: '["src"]'
```

The `--run` CLI arg takes precedence over `deploy.yaml`. The config name matches the instance name in SRC's WebUI (default: `src`).

## When `--run` Is Used

- **Skip Step 6** (clicking start) — SRC begins executing tasks immediately after the window renders
- **Verification** — capture the SRC window after launch. If the button changed to "停止", `--run` worked. If it still shows "启动", the config name is wrong — fall back to manual mode.
- **Completion detection** — preferred method: read the log file (see below). Fallback: visual inspection (queue shows "无任务")

## When `--run` Is NOT Used

Fall back to the original computer_use flow:
1. Wait for window (Step 5)
2. Click "启动" button (Step 6)
3. Monitor (Step 7)
4. Click "停止" button (Step 8)

## Log-Based Completion Detection

SRC writes task logs to a dated log file. You can check for completion without looking at the UI.

### Log file path

```
{install_dir}/log/{YYYY-MM-DD}_{config_name}.txt
```

For this machine:

```
~/scoop/apps/StarRailCopilot/current/log/{today}_{config_name}.txt
```

- `{today}` = current date in `YYYY-MM-DD` format (e.g. `2026-07-05`)
- `{config_name}` = instance name (default: `src`)

### Completion keywords

When all tasks are done and SRC enters the idle-waiting state, the log contains:

```
No task pending
Wait until YYYY-MM-DD HH:MM:SS for task `Restart`
```

### How to check

```python
# Read the last ~100 lines of today's log
tail -n 100 ~/scoop/apps/StarRailCopilot/current/log/2026-07-05_src.txt
```

Or use `grep` on the SRC log directory to check across multiple runs:

```bash
grep -l "No task pending" ~/scoop/apps/StarRailCopilot/current/log/*.txt
```

### When log file doesn't exist yet

The log file is only created after SRC has started running tasks. If the file doesn't exist, SRC hasn't started logging yet — wait a few seconds and retry. If it still doesn't appear, fall back to computer_use vision monitoring.

## Pitfalls

- **Clean environment required**: Always `cd` into install dir and clear `PYTHONPATH` before launching, even with `--run`. Otherwise import errors occur.
- **`--run` with wrong config name**: SRC starts but does nothing. **How to avoid:**
  1. Check `config/deploy.yaml` for the `Run` field — that's the config name
  2. Also check the `config/` directory for `.json` files with instance names
  3. Try `--run Alas` if `src` doesn't work
  4. After launching, capture the SRC window. If button still says "启动", the config name was wrong
  5. **Fallback:** kill SRC, relaunch without `--run`, click "启动" button manually
- **Still need `computer_use` to stop**: `--run` auto-starts but does NOT auto-stop. Still need to click "停止" (Step 8) when done.