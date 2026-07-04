---
name: openwrt-storage
description: OpenWrt USB extroot + swap via SSH/paramiko.
version: 0.2.0
author: Hermes
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [OpenWrt, Extroot, USB, Swap, Storage]
---

# OpenWrt 外挂存储扩容（extroot + swap）

## 适用场景

- "OpenWrt 内置 Flash 太小（8MB/16MB），装不了几个包"
- "给路由器加 swap 防 OOM 卡死"
- "外挂 USB 做 extroot overlay 扩容根文件系统"
- "路由器内存容易爆满卡死，需要超高 swappiness"

## Quick Reference

| 步骤         | 命令                                                                 | 用途          |
| ------------ | -------------------------------------------------------------------- | ------------- |
| 安装包       | `apk add block-mount e2fsprogs kmod-fs-ext4 parted kmod-usb-storage` | 必备工具      |
| 分盘         | `parted -s /dev/sda mklabel gpt` + `mkpart ...`                      | GPT 分区      |
| 格式化 ext4  | `mkfs.ext4 -F /dev/sda1`                                             | overlay 分区  |
| 格式化 swap  | `mkswap /dev/sda2`                                                   | swap 分区     |
| 复制 overlay | `tar -C /overlay -cvf - . \| tar -C /mnt -xf -`                      | 迁移数据      |
| 启用 extroot | `uci set fstab.@mount[-1].target='/overlay'`                         | 改挂载目标    |
| 启用 swap    | `uci set fstab.@swap[-1].enabled='1'`                                | 自动启用 swap |
| Swappiness   | `sysctl vm.swappiness=100` + 持久化                                  | 超高积极性    |
| Watermark    | `sysctl vm.watermark_scale_factor=1000` + 持久化                     | 更早回收      |

## 前置检查

- OpenWrt 内置 Flash 太小（如 8MB/16MB），装不了几个包
- 需要给路由器加 swap 防止 OOM 卡死
- 软路由/硬路由均适用（MT7620/MT7621/IPQ 系列等）

## 前置检查

```bash
# 查看 Flash/overlay 当前大小
df -h

# 查看是否已识别 U盘
ls /dev/sd*
fdisk -l /dev/sda

# 查看内存和 swap
free -m

# 确认包管理器
which apk  # OpenWrt 23.05+ 用 apk
which opkg # 旧版用 opkg
```

## 步骤

### 1. 安装必要包

```bash
apk update
apk add kmod-usb-storage fdisk e2fsprogs kmod-fs-ext4 block-mount parted
```

### 2. 重新分区 U盘

```bash
# 先卸载/关闭所有挂载
swapoff /dev/sda2 2>/dev/null
umount /dev/sda1 2>/dev/null

# 清除分区表（防止 kernel 缓存干扰）
dd if=/dev/zero of=/dev/sda bs=1M count=10
blockdev --rereadpt /dev/sda

# 重建分区
parted -s /dev/sda mklabel gpt
parted -s /dev/sda mkpart primary ext4 1MiB 3328MiB    # ~3.3GB ext4 for overlay
parted -s /dev/sda mkpart primary linux-swap 3328MiB 100%  # 剩余作 swap
```

### 3. 格式化

```bash
mkfs.ext4 -F /dev/sda1
mkswap /dev/sda2
```

### 4. 复制 overlay 数据到新分区

```bash
mount /dev/sda1 /mnt
tar -C /overlay -cvf - . | tar -C /mnt -xf -
umount /mnt
```

### 5. 配置 fstab（extroot + swap）

```bash
# 生成 fstab
block detect > /etc/config/fstab

# 用 uci 配置（比手动编辑更可靠）
uci set fstab.@mount[-1].target='/overlay'
uci set fstab.@mount[-1].enabled='1'
uci set fstab.@swap[-1].enabled='1'
uci commit fstab
```

### 6. 设置超高 swap 积极性

```bash
# 立即生效
sysctl vm.swappiness=100
sysctl vm.watermark_scale_factor=1000

# 持久化
echo 'vm.swappiness=100' >> /etc/sysctl.conf
echo 'vm.watermark_scale_factor=1000' >> /etc/sysctl.conf
```

### 7. 立即启用 swap + 重启激活 extroot

```bash
swapon /dev/sda2
reboot
```

### 8. 验证

```bash
df -h /overlay /         # 确认根目录已扩容
free -m                  # 确认 swap 启用
sysctl vm.swappiness     # 确认 swappiness=100
cat /proc/swaps          # swap 详情
block info | grep sda1   # 确认 sda1 挂载到 /overlay
```

## 常见踩坑 & 解决

### ❌ mkfs.ext4: "device is apparently in use by the system"

**原因**：kernel 缓存了旧分区表/文件系统，拒绝格式化。

**解决**：

1. 先 `swapoff /dev/sda2` + `umount /dev/sda1`
2. `dd if=/dev/zero of=/dev/sda bs=1M count=10` 擦除分区表
3. `blockdev --rereadpt /dev/sda` 刷新
4. 如果还不行 → **重启路由器**（`reboot`），重启后设备彻底干净
5. 重启后直接 `mkfs.ext4 -F /dev/sda1` 即可

### ❌ parted: "Partition(s) on /dev/sda are being used"

同样原因是 kernel 缓存。先 `dd` 擦除再 `mklabel gpt`，不行就重启。

### ❌ 重启后路由器长时间不上线

USB 文件系统损坏会导致启动时 kernel fsck 卡住很久（5min+）。

- 拔掉 U盘重启路由器
- 等路由器上线后再插 U盘
- 重新格式化

### ❌ block detect 没有生成 swap 条目

`block info` 确认 swap 已格式化。如果没有，手动添加到 fstab：

```bash
uci set fstab.@swap[-1]=swap
uci set fstab.@swap[-1].device='/dev/sda2'
uci set fstab.@swap[-1].enabled='1'
uci commit fstab
```

### ⚠️ U盘速度

MT7620 只有 USB 2.0，写入很慢（~5MB/s）。dd 大块写入（>50MB）会 SSH 超时，用小批量分批操作。

## SSH 连接方案

建议用 `paramiko`（Python）连 OpenWrt，避免系统没有 sshpass 的问题：

```python
import paramiko
client = paramiko.SSHClient()
client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
client.connect(host, 22, username, password, timeout=10,
               look_for_keys=False, allow_agent=False)

# 执行命令
stdin, stdout, stderr = client.exec_command("cmd", timeout=30)
out = stdout.read().decode('utf-8').strip()
```

## 安全机制

OpenWrt extroot 有自动回退机制：

- 如果 U盘启动失败，自动用内置 Flash overlay 启动
- 不会变砖
- 拔掉 U盘即可恢复原状
