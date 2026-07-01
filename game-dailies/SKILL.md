---
name: game-dailies
description: 自动代肝游戏日常
trigger: "当用户说 '帮我清日常'、'帮我代肝'、'帮我弄下游戏日常'、'该清体力了'"
---

# 游戏日常代肝自动化

## 工作流

### 1. 启动 MuMu 模拟器
```bash
"C:\Program Files\Netease\MuMu\nx_main\MuMuManager.exe" control -v 0 launch
"C:\Program Files\Netease\MuMu\nx_main\MuMuManager.exe" control -v 1 launch
```
sleep 20，无需检查 `is_android_started`

### 2. 验证 ADB
```bash
"C:\Program Files\Netease\MuMu\nx_main\adb.exe" connect 127.0.0.1:16384
"C:\Program Files\Netease\MuMu\nx_main\adb.exe" connect 127.0.0.1:16416
```

### 3. 后台启动 MAA + SRC
```bash
# MAA
"C:\Users\shiro\scoop\apps\maa\current\MAA.exe"

# SRC —— 必须 cd + 清 PYTHONPATH
cd "C:/Users/shiro/scoop/apps/StarRailCopilot/current" && env -u PYTHONPATH ./src.exe
```
两个都用 `background=true`

### 4. 操作 SRC
- `computer_use capture app='src' mode='som'` → 点击（"启动"按钮）→ 验证变为"停止"

### 5. 操作 MAA
- `computer_use capture app='MAA.exe' mode='som'` → 点击（"Link Start!"）→ 验证变为"停止"

## 踩坑

- **SRC vs MAA 用不同的 `app` 参数**：`src`（窗口标题 SRC）vs `MAA.exe`（标题每日变化，不可信）
- **SRC 必须清 PYTHONPATH**：hermes-agent 设了 PYTHONPATH 会污染 SRC 捆绑的 Python 3.10，导致 `exceptiongroup` 缺失。桌面双击无此问题。
- **按钮索引 #28 是位置规律**，若布局变化自适应 capture 后的实际结果