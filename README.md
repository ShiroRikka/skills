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
