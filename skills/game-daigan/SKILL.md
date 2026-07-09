---
name: game-daigan
description: >
  在 MuMu Android 模拟器上自动启动 MAA（明日方舟）和 SRC（星穹铁道）的日常代肝。
  当用户提及代肝、挂机、每日/周常、MAA、SRC 或模拟器自动挂机时激活。
  覆盖完整生命周期：识别实例 → 启动模拟器 → 启动代肝程序 → 触发自动化 → 等待完成 → 清理收尾。
license: MIT
compatibility: >
  Windows 专用。需要 MuMu Player 12、MAA 和/或 SRC（Scoop 安装）、
  以及 mumu-control 技能（本技能自动安装/更新）。
metadata:
  version: 2.0.0
  author: Hermes
  spec: agentskills-1.0
---

# 游戏自动代肝

在 MuMu Android 模拟器上使用 **MAA**（明日方舟）和 **SRC**（星穹铁道）自动完成日常。

> **本技能负责游戏自动化层。** 底层 MuMu 操作（启动、关闭、ADB、UI 自动化）委托给 `mumu-control`。

## 前置条件验证

操作前按顺序验证（任一失败则提示用户手动安装并中止）：

```bash
# 1. MuMu 是否安装（注册表检查）
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\MuMuPlayer" >nul 2>&1 && echo "MUMU_OK" || echo "MUMU_MISSING"

# 2. MAA（如果启用）
test -f "$HOME/scoop/apps/maa/current/MAA.exe" && echo "MAA_OK" || echo "MAA_MISSING"

# 3. SRC（如果启用）
test -f "$HOME/scoop/apps/StarRailCopilot/current/src.exe" && echo "SRC_OK" || echo "SRC_MISSING"
```

## 游戏选择

| 关键词 | 游戏 |
|-------|------|
| "明日方舟" / "Arknights" / "MAA" | → MAA |
| "星穹铁道" / "星铁" / "Star Rail" / "SRC" | → SRC |
| 未指定 / "都肝" / "全部" / "both" | → **两个都跑** |

## 流程

### 0. 准备 mumu-control

```bash
npx skills add https://skills.mumu.163.com/mumu-control --agent hermes-agent -g -y
skill_view(name='mumu-control')
pip install -r requirements.txt   # 在 MuMu 安装目录的 nx_main 下运行
python -m uiautomator2 init       # 每个模拟器实例只需一次
```

> 任一命令失败→重试1次→仍失败→提示用户手动运行并中止。

### 1. 识别模拟器

```bash
mumu-cli.exe info --vmindex all
```

解析 JSON，通过 `name` 字段找到目标实例：

| 游戏 | 预期名称 |
|------|---------|
| 明日方舟 | "明日方舟" |
| 星穹铁道 | "星穹铁道" |

> 无匹配时列出所有实例让用户确认。超时：30 秒。

### 2. 启动模拟器

按需要的索引并行启动（terminal + background）：

```bash
mumu-cli.exe control --vmindex <INDEX_1> launch
mumu-cli.exe control --vmindex <INDEX_2> launch
```

### 3. 等待就绪

每 10 秒轮询 `mumu-cli.exe info --vmindex all`，最多轮询 **15 次**（约 150 秒）。全部满足时立即停止：

- `is_process_started: true`
- `is_android_started: true`
- `player_state: "start_finished"`
- `adb_port` 字段出现

> 超时仍不满足 → 报告用户并中止。

就绪后隐藏窗口（避免遮挡后面代肝程序 UI）：

```bash
mumu-cli.exe control --vmindex <INDEX> hide_window
```

### 4. 启动代肝程序

通过 `terminal(background=true, notify_on_complete=true)` **并行**启动。

**MAA（如果启用）：**
```bash
"$HOME/scoop/apps/maa/current/MAA.exe"
```

**SRC（如果启用）— 需要干净环境：**
```bash
cd "$HOME/scoop/apps/StarRailCopilot/current" && env -u PYTHONPATH ./src.exe
```

> ⚠️ SRC 必须 `cd` 到安装目录并清除 `PYTHONPATH`，否则导入错误。

### 5. 验证代肝窗口

启动后等待 **10 秒**，然后用 `computer_use` 确认窗口就绪（**两个都要验证**）：

- **MAA：** `capture(mode='som', app='MAA')` — 确认"Link Start!"按钮存在
- **SRC：** `capture(mode='som', app='src')` — 确认"启动"按钮存在

> 必须用 `app=` 限定窗口。截图超时 15 秒 → 重试1次 → 失败则报告。

### 6. 开始代肝

**按顺序**处理 MAA → SRC。不要同时触发两个点击。

**MAA（如果启用）：**
1. `capture(mode='som', app='MAA')`
2. 按元素索引点击"Link Start!"
3. 确认按钮变为"停止"（重试上限 **2 次**，间隔 3 秒）

**SRC（如果启用）：**
1. `capture(mode='som', app='src')`
2. 按元素索引点击"启动"
3. 确认按钮变为"停止"（重试上限 **2 次**，间隔 3 秒）
4. 按钮不可见时先 `hide_window` 隐藏遮挡窗口后重试

### 7. 通知用户等待

> **"已经开始了！肝完了叫我来收尾就好~"**

等待用户回来说"肝完了"、"收尾"、"关了吧"、"done"。

### 8. 结束代肝

**按顺序**处理 MAA → SRC：

- **MAA：** 点击"停止" → 确认变为"Link Start!"（重试上限 2 次）
- **SRC：** 点击"停止" → 确认变为"启动"（重试上限 2 次；按钮不可见则先 `hide_window`）

### 9. 关闭模拟器

```bash
mumu-cli.exe control --vmindex <INDEX> shutdown
```

验证：轮询 `info` 直到 `is_process_started: false`（超时 30 秒）。

## 注意事项

### SRC 特有
- 必须 `cd` 到安装目录并清除 `PYTHONPATH`，否则导入报错

### 窗口管理
- MAA 窗口标题不稳定，用 `app='MAA'`（进程名）定位
- 窗口遮挡时先 `hide_window` 再重新截图
- 禁用坐标点击，必须用 `app=` + 元素索引

### 时序
- 后台启动 → 截图前等 10 秒
- 模拟器启动最坏 150 秒
- 按钮点击重试上限 2 次

### UI 假设
- 按钮文字依赖中文语言环境（Link Start! / 启动 / 停止）

### 多实例
- 两个代肝同时跑时，**必须顺序点击**按钮（共享元素索引时坐标会错乱）
- 截取目标窗口必须用 `app=` 过滤（全屏截图无法获取 SOM 覆盖层）

## 错误恢复

| 故障场景 | 处理方式 |
|---------|---------|
| MuMu 未安装 | 提示用户手动安装，中止 |
| 代肝程序缺失 | 提示用户通过 Scoop 安装，中止 |
| 模拟器启动超时 | 报错 + 建议检查 MuMu/ADB |
| 按钮点击失败 | 重试 2 次 → 报错 + 建议手动操作 |
| 代肝进程崩溃 | 检查 background 进程状态，重启一次 |
| SRC 导入错误 | 确认已 `cd` 到安装目录且清除 PYTHONPATH |