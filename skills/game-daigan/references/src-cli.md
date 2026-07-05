# StarRailCopilot (SRC) CLI 及配置参考

> ⚠️ **`--run` 参数在实际使用中不可靠** — 它经常无法匹配配置名称，也不会自动启动自动化。本 `game-daigan` 技能**不使用** `--run`；始终不加参数启动 SRC，然后在 UI 中点击"启动"按钮。本文档记录 `--run` 仅供参考，但建议使用手动点击方式。

SRC 支持在启动时自动运行任务，无需通过 `computer_use` 手动点击按钮。

## 启动时自动运行（两种方式）

### 方式一：CLI `--run` 参数（推荐）

在启动时传入配置名称：

```bash
cd ~/scoop/apps/StarRailCopilot/current && env -u PYTHONPATH ./src.exe --run src
```

或通过 Python：

```bash
cd ~/scoop/apps/StarRailCopilot/current && env -u PYTHONPATH python gui.py --run src
```

多个配置：`--run src src2`

### 方式二：`config/deploy.yaml` 配置

编辑 SRC 安装目录下的 `config/deploy.yaml`：

```yaml
Webui:
  # --run. 启动时自动运行指定配置
  # 'null' 默认不指定配置
  # '["src"]' 指定 "src" 配置
  # '["src","src2"]' 指定 "src" "src2" 配置
  Run: '["src"]'
```

`--run` CLI 参数优先级高于 `deploy.yaml`。配置名称与 SRC WebUI 中的实例名称一致（默认：`src`）。

## 使用 `--run` 时

- **跳过第 6 步**（点击启动）— SRC 窗口渲染后立即开始执行任务
- **验证** — 启动后截图 SRC 窗口。如果按钮变为"停止"，说明 `--run` 生效。如果仍显示"启动"，则配置名称错误 — 回退到手动模式。
- **完成检测** — 推荐方法：读取日志文件（见下文）。备选方案：视觉检查（队列显示"无任务"）

## 不使用 `--run` 时

回退到原始的 computer_use 流程：
1. 等待窗口出现（第 5 步）
2. 点击"启动"按钮（第 6 步）
3. 监控进度（第 7 步）
4. 点击"停止"按钮（第 8 步）

## 基于日志的完成检测

SRC 会将任务日志写入带日期的日志文件。可以无需查看 UI 即可检查完成状态。

### 日志文件路径

```
{install_dir}/log/{YYYY-MM-DD}_{config_name}.txt
```

对于本机：

```
~/scoop/apps/StarRailCopilot/current/log/{today}_{config_name}.txt
```

- `{today}` = 当前日期，格式 `YYYY-MM-DD`（例如 `2026-07-05`）
- `{config_name}` = 实例名称（默认：`src`）

### 完成关键词

当所有任务完成且 SRC 进入空闲等待状态时，日志中包含：

```
No task pending
Wait until YYYY-MM-DD HH:MM:SS for task `Restart`
```

### 如何检查

```bash
# 读取当天日志的最后约 100 行
tail -n 100 ~/scoop/apps/StarRailCopilot/current/log/2026-07-05_src.txt
```

或使用 `grep` 在 SRC 日志目录中跨多次运行检查：

```bash
grep -l "No task pending" ~/scoop/apps/StarRailCopilot/current/log/*.txt
```

### 日志文件尚未创建时

日志文件仅在 SRC 开始运行任务后才会创建。如果文件不存在，说明 SRC 尚未开始记录 — 等待几秒后重试。如果仍不出现，回退到 computer_use 视觉监控。

## 注意事项

- **需要干净环境**：即使使用 `--run`，也要始终 `cd` 到安装目录并清除 `PYTHONPATH` 后再启动。否则会出现导入错误。
- **`--run` 配置名称错误**：SRC 启动但什么都不做。**避免方法：**
  1. 检查 `config/deploy.yaml` 中的 `Run` 字段 — 那就是配置名称
  2. 同时检查 `config/` 目录中实例名称对应的 `.json` 文件
  3. 如果 `src` 不行，尝试 `--run Alas`
  4. 启动后，截图 SRC 窗口。如果按钮仍显示"启动"，说明配置名称错误
  5. **回退方案：** 杀掉 SRC，重新启动（不加 `--run`），手动点击"启动"按钮
- **仍需 `computer_use` 来停止**：`--run` 自动开始但**不会**自动停止。完成后仍需点击"停止"（第 8 步）。