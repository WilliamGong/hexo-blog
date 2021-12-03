---
title: LFS 搭建 6 系统配置
date: 2021-12-04T00:46:33+08:00
toc: true
cover:
thumbnail:
categories:
    - LFS
tags:
    - Linux
    - LFS
---
现在正式进入系统配置阶段。
# 网络配置
这里使用 systemd-networkd 和 systemd-resolved    
因为我需要远程登录，因此此处选择静态 ip：
## 静态 IP 配置
这里还不知道目标机器网卡接口名，这个值可以用于大部分有线接口，其他参数自行替换。
```
cat > /etc/systemd/network/10-eth-static.network << "EOF"
[Match]
Name=en*

[Network]
Address=192.168.1.105/24
Gateway=192.168.1.1
DNS=192.168.1.1
EOF
```
## 创建 /etc/resolv.conf 文件
创建一个符号链接：

    ln -sfv /run/systemd/resolve/resolv.conf /etc/resolv.conf
## 主机名
创建`/etc/hostname`：

    echo lfs > /etc/hostname
创建`/etc/hosts`：
```
cat > /etc/hosts << "EOF"
# Begin /etc/hosts

127.0.0.1 localhost.localdomain localhost
192.168.1.105 lfs.williamgong.io    lfs
::1       localhost ip6-localhost ip6-loopback
ff02::1   ip6-allnodes
ff02::2   ip6-allrouters

# End /etc/hosts
EOF
```
其中写着内网 ip 的那行自行修改。
# 时钟设置
这里使用硬件时钟，在最终重启后运行：

    timedatectl set-local-rtc 1
# Locale
创建`/etc/locale.conf`：
```
cat > /etc/locale.conf << "EOF"
LANG=en_US.utf8
EOF
```
# 创建 /etc/inputrc 文件 
```
cat > /etc/inputrc << "EOF"
# Begin /etc/inputrc
# Modified by Chris Lynn <roryo@roryo.dynup.net>

# Allow the command prompt to wrap to the next line
set horizontal-scroll-mode Off

# Enable 8bit input
set meta-flag On
set input-meta On

# Turns off 8th bit stripping
set convert-meta Off

# Keep the 8th bit for display
set output-meta On

# none, visible or audible
set bell-style none

# All of the following map the escape sequence of the value
# contained in the 1st argument to the readline specific functions
"\eOd": backward-word
"\eOc": forward-word

# for linux console
"\e[1~": beginning-of-line
"\e[4~": end-of-line
"\e[5~": beginning-of-history
"\e[6~": end-of-history
"\e[3~": delete-char
"\e[2~": quoted-insert

# for xterm
"\eOH": beginning-of-line
"\eOF": end-of-line

# for Konsole
"\e[H": beginning-of-line
"\e[F": end-of-line

# End /etc/inputrc
EOF
```
# 创建 /etc/shells 文件 
```
cat > /etc/shells << "EOF"
# Begin /etc/shells

/bin/sh
/bin/bash

# End /etc/shells
EOF
```
# systemd 相关
限制核心转储使用的最大磁盘空间（可选）：
```
mkdir -pv /etc/systemd/coredump.conf.d

cat > /etc/systemd/coredump.conf.d/maxuse.conf << EOF
[Coredump]
MaxUse=5G
EOF
```
# /etc/fstab
```
cat > /etc/fstab << "EOF"
# Begin /etc/fstab
# file system mount-point type options dump fsck
# order
/dev/sda3 / ext4 noatime,defaults 1 1
/dev/sda2 swap swap pri=1 0 0
/dev/sda1 /boot/efi vfat defaults 0 1
# End /etc/fstab
EOF
```
这是我这台机器上的，参数需要自行修改。     

至此，配置阶段结束，接下来就是编译内核了。