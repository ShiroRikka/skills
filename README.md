# Skills

## Hermes-Agent

### 将整个仓库添加为源

`hermes skills tap add ShiroRikka/skills`

这条命令把 `github.com/ShiroRikka/skills` 这个仓库添加为 Hermes 的**技能源**。之后你就可以：

```bash
hermes skills search openwrt
hermes skills install openwrt-storage   # 从你的 tap 安装
```

本质上就是告诉 Hermes："这个 GitHub 仓库里放着技能，去那里找。"

Hermes 的技能安全扫描分四个信任等级：

| 等级        | 来源             | 扫描                       |
| ----------- | ---------------- | -------------------------- |
| `builtin`   | Hermes 自带      | 不扫描                     |
| `official`  | 官方仓库         | 不扫描                     |
| `trusted`   | **你添加的 tap** | 走宽松扫描或跳过           |
| `community` | 公开技能市场     | 严格扫描，DANGEROUS 直接拦 |

### 或者

- 预览SKILL:`hermes skills inspect <identifier>`
- 安装SKILL:`hermes skills install <identifier>`
- `hermes skills search <query>` to search deeper

#### 例如

```bash
hermes skills inspect ShiroRikka/skills/game-dailies
```

```bash
hermes skills install ShiroRikka/skills/game-dailies
```

通过skills.sh安装/更新

```bash
npx skills add https://skills.mumu.163.com/mumu-control --agent hermes-agent -g -y
```

## game-daigan 与 mumu-control 的关系

`skills/game-daigan`（游戏自动代肝）依赖 `mumu-control`（MuMu 模拟器控制）技能。

- `game-daigan` 提供**游戏代肝逻辑** — 启动 MAA/SRC、验证窗口、点击开始/停止
- `mumu-control` 提供**模拟器底层操作** — 启动/关闭模拟器、ADB、UI 自动化点击/滑动/OCR

运行 `game-daigan` 时，agent 会自动先执行 `mumu-control` 的安装/更新（通过上述 `npx skills add` 命令），然后加载 `mumu-control` 技能处理模拟器相关步骤。

如果你单独使用 `mumu-control` 管理模拟器（不代肝），也可以直接用上述命令安装。

### 首次使用前

确保 `mumu-control` 已就绪：

```bash
# 1. 安装/更新 mumu-control
npx skills add https://skills.mumu.163.com/mumu-control --agent hermes-agent -g -y

# 2. 确认安装成功
hermes skills list | grep mumu-control
```
