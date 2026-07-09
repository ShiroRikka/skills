---
name: game-daigan
description: >
  在 MuMu Android 模拟器上启动自动日常游戏代肝（挂机）。
  当用户提到代肝/挂机/每日/日常/周常/daily tasks、
  想运行 MAA（明日方舟）或 SRC（星穹铁道）、
  或要求启动/监控/停止模拟器自动挂机时激活。
  涵盖完整生命周期：识别实例 → 启动模拟器 → 启动代肝程序 → 触发自动化 → 清理收尾。
license: MIT
compatibility: >
  Windows 专用。需要 MuMu Player 12 / MuMu、
  MAA（明日方舟）和/或 SRC（星穹铁道）通过 Scoop 安装、
  以及 mumu-control 技能（本技能自动安装/更新）。
metadata:
  version: 1.5.0
  author: Hermes
  spec: agentskills-1.0
---

# 游戏自动代肝

在 MuMu Android 模拟器上使用 **MAA**（明日方舟）和 **SRC**（星穹铁道）自动完成日常游戏任务。

> **本技能负责游戏自动化层。** 底层 MuMu 操作（启动、关闭、ADB、UI 自动化）委托给 `mumu-control` 技能。

**不涵盖：** 安装 MuMu 模拟器或代肝程序、配置代肝任务设置、常规 MuMu 管理。

## 何时激活

- "帮我代肝明日方舟" / "帮我代肝星铁"
- "启动模拟器并开始挂机" / "帮我把每日/周常做了"
- 任何涉及代肝、挂机、每日、MAA、SRC 或 MuMu 自动挂机的请求

## 前置条件

- **MuMu** 已安装（注册表 `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\MuMuPlayer`）
- **MAA**（明日方舟）：`$HOME/scoop/apps/maa/current/MAA.exe`
- **SRC**（星穹铁道）：`$HOME/scoop/apps/StarRailCopilot/current/src.exe`
- **`mumu-control` 技能**：见步骤 0，自动安装/更新

## 工作流概览

```
0. 安装/更新 mumu-control
1. 识别模拟器          → mumu-cli info --vmindex all
2. 启动模拟器          → mumu-cli control launch
3. 等待就绪            → 轮询 info 直到 is_android_started
4. 启动代肝程序        → MAA.exe / src.exe（后台）
5. 验证代肝窗口        → computer_use 确认 UI 可见
6. 开始自动化          → 点击按钮（Link Start! / 启动）
7. 通知用户            → "肝完了叫我来收尾~"
8. （用户通知完成）     → 停止代肝进程
9. 关闭模拟器          → mumu-cli control shutdown
```

**默认行为：** 未指定游戏时同时运行 MAA 和 SRC。只指定一个则运行指定。

## 流程

### 游戏选择

| 关键词 | 游戏 |
|-------|------|
| "明日方舟" / "Arknights" / "MAA" | → MAA |
| "星穹铁道" / "星铁" / "Star Rail" / "SRC" | → SRC |
| 未指定 / "都肝" / "全部" / "both" | → **两个都跑** |

### 0. 准备 mumu-control

开始操作前，确保 `mumu-control` 已安装且是最新版：

```bash
npx skills add https://skills.mumu.163.com/mumu-control --agent hermes-agent -g -y
```

加载技能，然后安装依赖：

```bash
skill_view(name='mumu-control')
pip install -r requirements.txt   # 在 MuMu 安装目录的 nx_main 下运行
python -m uiautomator2 init       # 每个模拟器实例只需一次
```

> 命令失败时重试一次，仍失败则提示用户手动运行。

### 1. 识别模拟器

```bash
mumu-cli.exe info --vmindex all
```

解析 JSON，通过 `name` 字段找到目标实例：

| 游戏 | 预期名称 |
|------|---------|
| 明日方舟 | "明日方舟" |
| 星穹铁道 | "星穹铁道" |

不匹配时列出所有实例让用户确认。

### 2. 启动模拟器

```bash
mumu-cli.exe control --vmindex <INDEX_1> launch
mumu-cli.exe control --vmindex <INDEX_2> launch
```

### 3. 等待就绪

每 10 秒轮询 `mumu-cli.exe info --vmindex all`，直到所有目标索引满足：
- `is_process_started: true`
- `is_android_started: true`
- `player_state: "start_finished"`
- `adb_port` 字段出现

**全部满足后立即停止轮询**（典型等待 30–120s）。

隐藏模拟器窗口（避免遮挡后面代肝程序的 UI）：

```bash
mumu-cli.exe control --vmindex <INDEX> hide_window
```

### 4. 启动代肝程序

通过 `terminal(background=true)` **并行**启动。

**MAA（如果启用）：**
```bash
"$HOME/scoop/apps/maa/current/MAA.exe"
```

**SRC（如果启用）** — 需要干净环境：
```bash
cd "$HOME/scoop/apps/StarRailCopilot/current" && env -u PYTHONPATH ./src.exe
```

> ⚠️ SRC 必须 `cd` 到安装目录并清除 `PYTHONPATH`，否则导入错误。

### 5. 验证代肝窗口

等待 5–15 秒，用 `computer_use` 确认每个程序的 UI 可见（**两个都要验证**）：

- **MAA：** `computer_use(action='capture', mode='som', app="MAA")` — 确认"Link Start!"按钮存在
- **SRC：** `computer_use(action='capture', mode='som', app="src")` — 确认"启动"按钮存在

> ⚠️ 必须使用 `app=` 参数限定窗口，不要全屏截图。

### 6. 开始代肝

**按顺序**处理 MAA 和 SRC — 完成一个再开始下一个。不要同时触发两个点击。

**MAA（如果启用）：**
1. 截图获取 SOM 覆盖层
2. 按元素索引点击"Link Start!"
3. 确认按钮变为"停止"（否则重试一次）

**SRC（如果启用）：**
1. 截图获取 SOM 覆盖层
2. 按元素索引点击"启动"
3. 确认按钮变为"停止"（否则重试一次）
4. 按钮不可见时先 `hide_window` 隐藏遮挡窗口

### 7. 通知用户

> **"已经开始了！肝完了叫我来收尾就好~"**

等待用户回来说"肝完了"、"收尾"、"关了吧"、"done"。

### 8. 结束代肝

**按顺序**处理 MAA 和 SRC：

**MAA：** 点击"停止" → 确认变为"Link Start!"（否则重试一次）
**SRC：** 点击"停止" → 确认变为"启动"（否则重试一次，按钮不可见时先 `hide_window`）

### 9. 关闭模拟器

```bash
mumu-cli.exe control --vmindex <INDEX> shutdown
```

验证：`mumu-cli.exe info --vmindex all` — 确认 `is_process_started: false`。

## 注意事项

- **SRC 需干净环境**：始终 `cd` 到安装目录并清除 `PYTHONPATH`
- **MAA 窗口标题不稳定**：不要匹配标题，用 `app="MAA"` 基于进程名定位
- **后台启动 ≠ 立即就绪**：截图前等待 5–15 秒
- **按钮文字依赖语言环境**：本技能假设中文 UI
- **窗口遮挡**：按钮不可见时先 `hide_window` 隐藏遮挡窗口，再重新截图
- **必须顺序点击**：不要同时点击两个代肝按钮（共享元素索引时坐标会错乱）
- **禁用坐标点击**：必须用 `computer_use(action='click', element=<N>, app="<PROCESS>")` 按元素索引点击
- **截取目标窗口必须用 `app=` 过滤**：不加 `app=` 的全屏截图无法获取 SOM 覆盖层