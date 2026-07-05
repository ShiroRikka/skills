---
name: game-daigan
description: >
  在 MuMu Android 模拟器上启动自动日常游戏代肝（挂机）。
  当用户提到代肝/挂机/每日/日常/周常/daily tasks，
  想运行 MAA（明日方舟）或 SRC（星穹铁道），
  或要求启动/监控/停止模拟器自动挂机时激活此技能。
  涵盖完整的代肝生命周期：识别实例 → 启动模拟器 → 启动代肝程序 → 点击开始 →
  监控日志完成 → 清理收尾。
license: MIT
compatibility: >
  Windows 专用。需要 MuMu Player 12 已安装，MAA（明日方舟）和/或
  SRC/StarRailCopilot（星穹铁道）通过 Scoop 安装，并且 mumu-control
  技能可用。
metadata:
  version: 1.2.0
  author: Hermes
  spec: agentskills-1.0
---

# 游戏自动代肝

在 MuMu Android 模拟器上使用 **MAA**（明日方舟）和 **SRC**（星穹铁道）辅助程序自动完成日常游戏任务。涵盖完整的代肝生命周期：识别模拟器实例 → 通过 mumu-control 启动 → 启动代肝程序 → 触发自动化 → 监控完成状态 → 清理收尾。

> **技能结构说明（渐进式文档）：**
>
> - 本 `SKILL.md` 是核心流程 — 执行代肝任务时加载此文件
> - **只有**需要基于日志检测 MAA 完成状态时，再阅读 [references/maa-log.md](references/maa-log.md)
> - **只有** SRC 启动遇到问题时，再阅读 [references/src-cli.md](references/src-cli.md)

> **本技能负责游戏自动化（代肝）层。** 对于底层的 MuMu 模拟器操作（启动、关闭、ADB、UI 自动化），委托给 `mumu-control` 技能处理 — 详见 [与 mumu-control 的关系](#与-mumu-control-的关系)。

**不涵盖：** 安装模拟器或代肝程序、在代肝界面内配置游戏任务设置、或常规的 MuMu Player 12 管理。

## 何时激活

- "帮我代肝明日方舟" / "帮我代肝星铁"
- "启动模拟器并开始挂机"
- "帮我把每日/周常做了"
- 用户要求启动、监控或停止基于模拟器的游戏自动化
- 任何涉及代肝、挂机、每日、MAA、SRC 或 MuMu 自动挂机的请求

## 前置条件

- **MuMu Player 12** 已安装（`mumu-control` 会自动检测路径）
- **MAA**（明日方舟）通过 Scoop 安装：`$HOME/scoop/apps/maa/current/MAA.exe`
- **SRC / StarRailCopilot**（星穹铁道）通过 Scoop 安装：`$HOME/scoop/apps/StarRailCopilot/current/src.exe`
- **`mumu-control` 技能** 在同一技能目录下可用（`skills/mumu-control/SKILL.md`）

## 与 mumu-control 的关系

官方 [`mumu-control`](../mumu-control/SKILL.md) 技能涵盖所有 MuMu Player 12 操作：

| 操作                         | 由谁处理                                                 |
| ---------------------------- | -------------------------------------------------------- |
| 查找安装路径                 | `mumu-control` — 自动检测链                              |
| 列出模拟器实例（`info`）     | `mumu-control` — `mumu-cli.exe info --vmindex all`       |
| 启动/关闭模拟器              | `mumu-control` — `control launch` / `control shutdown`   |
| 等待启动完成                 | `mumu-control` — 轮询 `player_state == start_finished`   |
| ADB 连接及命令               | `mumu-control` — 内置 `adb.exe`、应用管理                |
| UI 自动化（点击、滑动、OCR） | `mumu-control` — Python 脚本（`tap.py`、`uiscan.py` 等） |

**本技能（`game-daigan`）只添加游戏相关的知识** — 哪个如何以正确的参数和环境启动 MAA/SRC、如何验证它们正在运行等。

执行以下流程时，加载 `mumu-control` 技能处理模拟器相关步骤，并查阅其参考文档。

## 工作流概览

```
1. 识别模拟器          → mumu-cli info --vmindex all（mumu-control）
2. 启动模拟器          → mumu-cli control launch（mumu-control）
3. 等待就绪            → 轮询 info 直到 is_android_started（mumu-control）
4. 启动代肝程序        → MAA.exe / src.exe（game-daigan）
5. 验证代肝窗口        → 确认 UI 可见（game-daigan）
6. 开始自动化          → 点击按钮（game-daigan）
7. 通知用户            → "已经开始代肝了，完成后叫我"
8. （用户通知完成）    → 杀掉代肝进程（game-daigan）
9. 关闭模拟器          → mumu-cli control shutdown（mumu-control）
```

**默认行为：** 如果用户未指定游戏，**同时运行** MAA 和 SRC。如果只指定了一个，则只运行指定的那个。

## 流程

### 游戏选择

解析用户的请求：

| 提到关键词                                | 游戏           |
| ----------------------------------------- | -------------- |
| "明日方舟" / "Arknights" / "MAA"          | → MAA          |
| "星穹铁道" / "星铁" / "Star Rail" / "SRC" | → SRC          |
| 未指定 / "都肝" / "全部" / "both"         | → **两个都跑** |

---

### 1. 识别模拟器

查阅 `mumu-control` 技能（阅读 `skills/mumu-control/SKILL.md`）获取完整的 `mumu-cli.exe` 命令参考。运行：

```bash
mumu-cli.exe info --vmindex all
```

解析 JSON。通过匹配 `"name"` 字段找到每个目标游戏的 `--vmindex`：

| 游戏     | 预期名称   |
| -------- | ---------- |
| 明日方舟 | "明日方舟" |
| 星穹铁道 | "星穹铁道" |

如果 JSON 中的名称不完全匹配，列出所有实例并让用户确认哪个索引对应哪个游戏。

---

### 2. 启动模拟器

使用 `mumu-control`：

```bash
# 启动每个目标模拟器
mumu-cli.exe control --vmindex <INDEX_1> launch
mumu-cli.exe control --vmindex <INDEX_2> launch
```

---

### 3. 等待就绪

每 10 秒轮询一次 `mumu-cli.exe info --vmindex all`，直到每个目标索引**全部**满足以下条件：

- `"is_process_started": true`
- `"is_android_started": true`
- `"player_state": "start_finished"`
- `"adb_port"` 字段出现

**重要：** 一旦所有目标满足就绪条件，**立即停止轮询** — 不要继续固定的轮询次数。典型等待时间：每个模拟器 30–120 秒。两个模拟器独立启动 — 每次轮询完整列表。

**现在隐藏所有模拟器窗口**，以免后续挡住代肝程序界面：

```bash
mumu-cli.exe control --vmindex <INDEX_1> hide_window
mumu-cli.exe control --vmindex <INDEX_2> hide_window
```

代肝程序（MAA/SRC）通过 ADB 与模拟器通信 — 模拟器窗口不需要可见。

---

### 4. 启动代肝程序

通过 `terminal(background=true)` **并行**启动代肝程序。

**MAA（如果启用了）：**

```bash
"$HOME/scoop/apps/maa/current/MAA.exe"
```

> 使用显式完整路径（`$HOME/scoop/apps/maa/current/MAA.exe`）而不是依赖 PATH 中的 `MAA.exe` — 在 git-bash 中运行时更可靠。

直接启动 — 窗口打开后进入第 6 步点击"Link Start!"。

**SRC（如果启用了）：**

```bash
cd "$HOME/scoop/apps/StarRailCopilot/current" && env -u PYTHONPATH ./src.exe
```

> ⚠️ SRC 需要干净的环境：始终先 `cd` 进入安装目录并清除 `PYTHONPATH`。从其他目录运行或使用被污染的 `PYTHONPATH` 会导致导入错误。

窗口打开后进入第 6 步手动点击"启动"按钮。

---

### 5. 验证代肝窗口

启动后等待 5–15 秒，然后用 `computer_use` 确认每个程序的 UI 可见。

**MAA：** `computer_use(action='capture', app="MAA")` — 验证主 UI 标签页（一键长草、自动战斗、小工具、设置）和"Link Start!"按钮。

**SRC：** `computer_use(action='capture', app="src")` — 验证调度器 UI（左侧导航、日志区域）和"启动"按钮。

> 💡 如果目标窗口的按钮在截图中不可见，可能是其他窗口（模拟器、MAA 等）挡住了它。先使用 mumu-control 隐藏或最小化遮挡窗口：`mumu-cli.exe control --vmindex <INDEX> hide_window`，然后重新截图。

---

### 6. 开始代肝

**按顺序**处理 MAA 和 SRC — 彻底完成一个再开始下一个。不要同时触发两个点击；每个点击都必须验证通过后才能继续。

**MAA（如果启用了）：**

1. `computer_use(action='capture', mode='som', app="MAA")` — 获取编号覆盖层
2. 按元素索引点击"Link Start!"按钮
3. 重新截图确认按钮文字变为"停止"
   - 如果仍然是"Link Start!"，重试点击 + 重新截图一次
4. 确认 MAA 正在运行后，再处理 SRC

**SRC（如果启用了）：**

1. `computer_use(action='capture', mode='som', app="src")` — 获取编号覆盖层
2. 按元素索引点击"启动"按钮
3. 如果按钮在截图中不可见，可能是其他窗口挡住了。先隐藏模拟器窗口：`mumu-cli.exe control --vmindex <INDEX> hide_window`，然后重新截图并点击
4. 重新截图确认按钮文字变为"停止"
   - 如果仍然是"启动"，重试点击 + 重新截图一次

---

### 7. 通知用户

所有代肝程序正在运行。告诉用户：

> **"已经开始了！肝完了叫我来收尾就好~"**

在此停止。等待用户回来说"肝完了"、"结束了"、"收尾"、"关了吧"、"done"之类的话。

---

### 8. 关闭模拟器

使用 `mumu-control`：

```bash
mumu-cli.exe control --vmindex <INDEX_1> shutdown
mumu-cli.exe control --vmindex <INDEX_2> shutdown
```

验证：`mumu-cli.exe info --vmindex all` — 确认每个目标显示 `"is_process_started": false`。

## 注意事项（避坑指南）

- **SRC 需要干净环境**：始终 `cd` 到安装目录并清除 `PYTHONPATH` 后再启动。从其他位置运行或使用被污染的 `PYTHONPATH` 会导致导入错误。
- **MAA 窗口标题不稳定**：它包含版本/构建信息，每次更新都会变化。永远不要匹配窗口标题 — 使用 `app="MAA"` 基于进程名定位。
- **后台启动 ≠ 立即就绪**：代肝程序需要 5–30 秒渲染出第一个窗口。截图前先等待。
- **多实例歧义**：如果多个模拟器可能匹配游戏名称，检查 `"is_main"` 或询问用户。
- **按钮文字依赖语言环境**：本技能假设中文 UI（MAA/SRC 默认）。如果语言环境不同，按钮文字会不一样。
- **窗口遮挡**：SRC 的"启动"按钮或 MAA 的"Link Start!"可能被其他窗口（模拟器、MAA 自身等）挡住。如果按钮在截图中不可见，先使用 `mumu-cli.exe control --vmindex <INDEX> hide_window` 隐藏遮挡窗口，再重新截图。这通常是点击失败的真实原因，而不是目标窗口类型问题。
- **必须顺序点击**：不要同时点击两个代肝按钮。当两个窗口共享相同的元素索引时（例如两个窗口的启动按钮都是元素 28），同时触发两次点击会导致坐标解析使用错误的窗口映射。始终先点一个，验证完成，再点下一个。

## 验证循环（收尾确认）

关闭模拟器后，执行验证并根据需要重试：

1. **验证模拟器已关闭：** `mumu-cli.exe info --vmindex all` — 确认每个目标显示 `"is_process_started": false`
2. **检查通过后**才报告完成
