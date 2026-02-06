---
title: 香橙派5 Pro UMS刷机全流程（含进度条）
tags:
  - 香橙派5Pro
  - RK3588
  - U-Boot
  - UMS
  - NoMachine
  - Headless
  - 刷机
  - dd
  - pv
categories:
  - - 技术实践
  - - 系统工程
  - - 系统编程
  - - 嵌入式开发
date: 2026-02-06 20:55:00
---

这篇是给未来的自己留的完整流程：当香橙派5 Pro（或其他 RK3588 板子）系统内核玩坏后，如何通过 **U-Boot 的 UMS 模式**，在电脑侧用 `dd` 直接重刷整盘镜像。

<!-- more -->

## 一、准备条件

你需要：

1. 一根 USB 转串口线（任意能进 UART 控制台即可）。
2. 板子和主机连接线（常见是 USB-A to C 或 C to C，取决于板子接口）。
3. 目标 `.img` 镜像文件（不是 `.xz`，必要时先解压）。
4. Linux 主机的 root 权限（`sudo`）。

> 这套流程会直接覆盖目标设备分区表和所有分区，请确认目标盘无重要数据。

## 二、进入 U-Boot 并切换 UMS

1. 串口连上板子，打开串口终端。
2. 板子上电或按 `reset`。
3. 在启动日志滚动时按 `Ctrl + C`，打断自动启动，进入 U-Boot shell。
4. 先执行：

```bash
ums
```

看命令提示后，再执行常用形式：

```bash
ums 0 mmc 0
```

你给出的原始帮助如下（关键点是 `devtype` 默认 `mmc`）：

```text
ums - Use the UMS [USB Mass Storage]

Usage:
ums <USB_controller> [<devtype>] <dev[:part]>  e.g. ums 0 mmc 0
    devtype defaults to mmc
```

补充说明：

- `USB_controller` 通常是 `0`。
- `devtype` 可能是 `mmc` / `nvme` / 其他块设备类型，按板子实际为准。
- `dev` 通常是 `0`（第一块设备）。

## 三、主机侧确认设备节点

板子进入 UMS 后，主机上会出现新块设备（例如 `/dev/sda`）。先确认，避免误刷本机盘：

```bash
lsblk -o NAME,SIZE,MODEL,MOUNTPOINT
```

## 四、执行刷写（带进度条）

### 方案 A：`dd` 自带进度（最省事）

```bash
sudo dd if="/path/to/your-system.img" of=/dev/sdX bs=4M conv=fsync status=progress
sync
```

### 方案 B：`pv` + `dd`（显示更友好的进度/速率）

```bash
pv "/path/to/your-system.img" | sudo dd of=/dev/sdX bs=4M conv=fsync status=none
sync
```

如果你和我一样，镜像就在 `~/Downloads`，可以直接替换成实际路径：

```bash
pv "/home/cagedbird/Downloads/ubuntu-24.04-preinstalled-desktop-arm64-orangepi-5-pro.img" | sudo dd of=/dev/sda bs=4M conv=fsync status=none
sync
```

## 五、收尾

1. `dd` 结束并 `sync` 后，回到串口侧。
2. `Ctrl + C` 退出 UMS。
3. 断电重上电，或直接按 `reset`。
4. 板子应从新系统正常启动。

## 六、常见坑

### 1) 刷错盘

最危险的问题。一定先 `lsblk`，必要时拔掉其他移动硬盘再刷。

### 2) 设备节点变化

有时第一次是 `/dev/sda`，下一次可能变 `/dev/sdb`，不要硬记路径。

### 3) 镜像没解压

`dd` 的 `if=` 要用 `.img`。如果手里是 `.img.xz`，先解压再刷。

### 4) UMS 参数不对

先执行裸 `ums` 看帮助，再按板子实际存储介质改 `devtype`。

## 七、无显示器完成首启 OOBE（实战验证版）

这套流程已经实测打通，适用于 `joshua-riek` 这类首启进入 Ubuntu 图形初始化向导（OOBE）的镜像。

### 7.1 实战拓扑

1. 串口：`FT232`（比 CH340 稳定，推荐）。
2. UMS 线：USB 线一直连着，必要时可随时 `ums 0 mmc 0` 进“暴力修复”模式。
3. 网络：Arch 主机给板子共享网络 + RJ45 同网段互通。

### 7.2 关键流程（双路径并列）

#### A 路径：先打通 root SSH（这次最终成功主路径）

在 U-Boot 菜单先进入 rescue target，然后执行：

```bash
# 1) 设置 root 密码
passwd root

# 2) 允许 root 密码登录 SSH
vim /etc/ssh/sshd_config
```

确认至少包含：

```text
PermitRootLogin yes
PasswordAuthentication yes
PubkeyAuthentication yes
```

这次我直接 `reboot` 让配置生效（未单独 `systemctl restart ssh`）。

重启后从 Arch 直接：

```bash
ssh root@<board-ip>
```

#### B 路径：SSH 上板后安装 NoMachine 进图形 OOBE

```bash
apt update
apt install -y wget
wget https://download.nomachine.com/download/8.16/Arm/nomachine_8.16.1_1_arm64.deb
apt install -y ./nomachine_8.16.1_1_arm64.deb
```

实测坑点：安装完直接重启，`nxserver` 可能不自动拉起，NoMachine 客户端会连不上。

需要手动强拉一次：

```bash
systemctl restart nxserver
# 或
/usr/NX/bin/nxserver --restart
```

之后在 Arch 的 NoMachine 客户端连接板子 IP（默认端口 4000），即可进入图形界面完成 OOBE 创建用户。

### 7.3 成功判据

1. `ssh root@<board-ip>` 可密码登录。
2. NoMachine 可连图形桌面。
3. OOBE 用户创建完成。
4. 重启后依然可用。

### 7.4 说明

- 这次没有继续修“正常模式串口交互不佳”的问题，因为 SSH + NoMachine 已满足开发可用性。
- 对开发板（非公网服务器）场景，临时放宽 root 登录策略是可接受的效率优先方案。
- 若后续外网使用，建议再收紧 SSH 策略。

## 八、简记（速查版）

串口侧：

```bash
ums 0 mmc 0
```

主机侧：

```bash
lsblk -o NAME,SIZE,MODEL,MOUNTPOINT
pv "/path/to/system.img" | sudo dd of=/dev/sdX bs=4M conv=fsync status=none
sync
```

一句话：`Ctrl + C` 进 U-Boot -> `ums 0 mmc 0` -> 确认 `/dev/sdX` -> `pv | dd` 覆盖写入 -> `sync` -> 重启。

---

这套方法的本质就是：**让板子在 U-Boot 阶段把本地存储“伪装”成 USB 大容量盘，再从主机整盘写入镜像**。对救砖、重装系统非常直接有效。
