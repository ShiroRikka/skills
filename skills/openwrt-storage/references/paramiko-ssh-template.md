# OpenWrt Paramiko 批量操作模板

## 基础连接模板

```python
import paramiko

host = "192.168.10.254"
username = "root"
password = "your_password"

def run_cmd(client, cmd, title=None, timeout=60):
    if title:
        print(f">>> {title}")
    print(f"$ {cmd}")
    stdin, stdout, stderr = client.exec_command(cmd, timeout=timeout)
    out = stdout.read().decode('utf-8').strip()
    err = stderr.read().decode('utf-8').strip()
    lines = out.split('\n')
    if len(lines) > 20:
        print('\n'.join(lines[-20:]))
    else:
        print(out)
    if err and ('ERROR' in err.upper() or 'error:' in err.lower()):
        print(f"[STDERR]: {err}")
    return out, err

client = paramiko.SSHClient()
client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
client.connect(host, 22, username, password, timeout=10,
               look_for_keys=False, allow_agent=False)

# ... 执行命令 ...

client.close()
```

## 等待路由器重启后重连

```python
import time

# 发送重启
client.exec_command("reboot", timeout=5)
client.close()

# 等待上线
for attempt in range(20):
    try:
        c = paramiko.SSHClient()
        c.set_missing_host_key_policy(paramiko.AutoAddPolicy())
        c.connect(host, 22, username, password, timeout=8,
                  look_for_keys=False, allow_agent=False)
        print(f"路由器已上线 (尝试{attempt+1})")
        # 继续操作...
        c.close()
        break
    except Exception:
        time.sleep(8)
```

## 关键参数

- `look_for_keys=False` — 不尝试密钥认证（纯密码连接）
- `allow_agent=False` — 不尝试 SSH agent
- timeout 设置：连接 10s，命令执行根据操作类型 15~120s
