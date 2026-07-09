---
name: game-daigan
description: "在 MuMu 模拟器上自动启动 MAA（明日方舟）和 SRC（星穹铁道）的日常代肝。当用户提及代肝、挂机、每日/周常、MAA、SRC 或模拟器自动挂机时激活。"
version: 2.1.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [gaming, automation, arknights, star-rail, mumu]
    related_skills: []
---

# 游戏自动代肝

在 MuMu Android 模拟器上使用 **MAA**（明日方舟）和 **SRC**（星穹铁道）自动完成日常。底层 MuMu 操作（启动、关闭、ADB、UI 自动化）委托给 `mumu-control` 技能。

## When to Use

| 关键词 | 目标 |
|-------|------|
| "明日方舟" / "Arknights" / "MAA" | → MAA |
| "星穹铁道" / "星铁" / "Star Rail" / "SRC" | → SRC |
| 未指定 / "都肝" / "全部" / "both" | → 两个都跑 |

## 前置条件

- Windows + MuMu Player 12
- MAA 和/或 SRC（通过 Scoop 安装）
- `mumu-control` 技能（本流程自动安装）

### 验证

```bash
# MuMu 安装
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\MuMuPlayer" >nul 2>&1 && echo MUMU_OK || echo MUMU_MISSING

# MAA（如果启用）
test -f "$HOME/scoop/apps/maa/current/MAA.exe" && echo MAA_OK || echo MAA_MISSING

# SRC（如果启用）
test -f "$HOME/scoop/apps/StarRailCopilot/current/src.exe" && echo SRC_OK || echo SRC_MISSING
```

任一缺失 → 提示用户手动安装并中止。

## 流程

### 0. 准备 mumu-control

```bash
npx skills add https://skills.mumu.163.com/mumu-control --agent hermes-agent -g -y
skill_view(name='mumu-control')
pip install -r requirements.txt
python -m uiautomator2 init
```

任一命令失败 → 重试 1 次 → 仍失败 → 提示用户手动运行并中止。

### 1. 识别模拟器

```bash
mumu-cli.exe info --vmindex all
```

解析 JSON，通过 `name` 字段匹配目标实例：

| 游戏 | 预期名称 |
|------|---------|
| 明日方舟 | "明日方舟" |
| 星穹铁道 | "星穹铁道" |

无匹配 → 列出所有实例让用户确认。超时：30 秒。

### 2. 启动模拟器

按索引并行启动（`terminal` + `background`）：

```bash
mumu-cli.exe control --vmindex <INDEX_1> launch
mumu-cli.exe control --vmindex <INDEX_2> launch
```

### 3. 等待就绪

每 10 秒轮询 `mumu-cli.exe info --vmindex all`，最多 **15 次**（约 150 秒）。全部满足时立即停止：

- `is_process_started: true`
- `is_android_started: true`
- `player_state: "start_finished"`
- `adb_port` 字段出现

超时 → 报错并中止。

就绪后隐藏窗口：

```bash
mumu-cli.exe control --vmindex <INDEX> hide_window
```

### 4. 启动代肝程序

通过 `terminal(background=true, notify_on_complete=true)` **并行**启动：

**MAA（如果启用）：**
```bash
"$HOME/scoop/apps/maa/current/MAA.exe"
```

**SRC（如果启用）：**
```bash
cd "$HOME/scoop/apps/StarRailCopilot/current" && env -u PYTHONPATH ./src.exe
```

> ⚠️ SRC 必须 `cd` 到安装目录并清除 `PYTHONPATH`，否则导入错误。

### 5. 验证代肝窗口

启动后等待 **10 秒**，用 `computer_use` 确认两个窗口就绪：

| 程序 | capture 参数 | 确认条件 |
|-----|-------------|---------|
| MAA | `app='MAA'` | "Link Start!" 按钮存在 |
| SRC | `app='src'` | "启动" 按钮存在 |

截图超时 15 秒 → 重试 1 次 → 失败则报告。

### 6. 开始代肝

**按顺序**处理 MAA → SRC。不要同时触发两个点击。

**MAA（如果启用）：**
1. `capture(mode='som', app='MAA')` → 点击 "Link Start!"
2. 确认按钮变为 "停止"（重试 2 次，间隔 3 秒）

**SRC（如果启用）：**
1. `capture(mode='som', app='src')` → 点击 "启动"
2. 确认按钮变为 "停止"（重试 2 次，间隔 3 秒）
3. 按钮不可见时先 `hide_window` 隐藏遮挡窗口后重试

### 7. 通知用户

> **"已经开始了！肝完了叫我来收尾就好~"**

等待用户说 "肝完了"、"收尾"、"关了吧"、"done"。

### 8. 结束代肝

**按顺序** MAA → SRC：

- **MAA：** 点击 "停止" → 确认变为 "Link Start!"（重试 2 次）
- **SRC：** 点击 "停止" → 确认变为 "启动"（重试 2 次；按钮不可见则先 `hide_window`）

### 9. 关闭模拟器

```bash
mumu-cli.exe control --vmindex <INDEX> shutdown
```

验证：轮询 `info` 直到 `is_process_started: false`（超时 30 秒）。

## 错误恢复

| 故障场景 | 处理方式 |
|---------|---------|
| MuMu 未安装 | 提示用户手动安装，中止 |
| 代肝程序缺失 | 提示用户通过 Scoop 安装，中止 |
| 模拟器启动超时 | 报错 + 建议检查 MuMu/ADB |
| 按钮点击失败 | 重试 2 次 → 报错 + 建议手动操作 |
| 代肝进程崩溃 | 检查 background 进程状态，重启一次 |
| SRC 导入错误 | 确认已 `cd` 到安装目录且清除 PYTHONPATH |

## Common Pitfalls

1. **MAA 窗口标题不稳定** — 用 `app='MAA'`（进程名）而非窗口标题定位
2. **窗口遮挡** — 先 `hide_window` 再重新截图；禁用坐标点击，必须用 `app=` + 元素索引
3. **按钮文字依赖中文语言环境** — "Link Start!" / "启动" / "停止" 在不同语言环境下可能不同
4. **共享元素索引冲突** — 两个代肝同时跑时，必须顺序点击按钮，不能用坐标
5. **SRC 导入错误** — 必须 `cd` 到安装目录并 `env -u PYTHONPATH`，否则报导入错误
6. **时序依赖** — 后台启动后等 10 秒再截图；模拟器启动最坏 150 秒；按钮点击重试上限 2 次

## Verification Checklist

- [ ] 前置条件验证通过（MuMu、MAA/SRC 均已安装）
- [ ] 模拟器识别正确，启动成功
- [ ] 模拟器就绪（四个字段全部满足）
- [ ] 代肝程序窗口就绪（"Link Start!" / "启动" 按钮可见）
- [ ] 开始按钮点击成功，状态切换为 "停止"
- [ ] 用户已收到通知
- [ ] 结束代肝后按钮恢复为 "Link Start!" / "启动"
- [ ] 模拟器已正常关闭