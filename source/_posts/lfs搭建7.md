---
title: LFS 搭建 7 内核与 GRUB
date: 2021-12-04T13:35:18+08:00
toc: true
cover:
thumbnail:
categories:
tags:
---

现在进入最后一部分，内核与 GRUB 的安装。
# 构建 Linux 内核
先解压：
```
tar -xvf linux-5.13.12.tar.xz
cd linux-5.13.12
```
清理源码树，虽然才解压没什么必要：

    make mrproper
## 配置
此处使用：

    make menuconfig
> 原来我一直想使用 arch 的配置然后`oldconfig`，但试了很多次后机器启动一直失败。

配置选项说明参见金步国的博客：http://www.jinbuguo.com/kernel/longterm-linux-kernel-options.html     
虽然是 4.4 的，但大部分选项都没变，尤其是驱动相关。     
尽量不要改默认的配置，驱动相关的另说。     
下面是具体的配置选项：   
General setup 下：
```  
[*] Control Group support  --->
# 手册里没有，但 systemd 建议打开
[*] Checkpoint/restore support

[*] Configure standard kernel features (expert users)  --->
# 子选项需要关闭以下几个：
    [ ]   Enable 16-bit UID system calls
    [ ]   sgetmask/ssetmask syscalls support
    [ ]   Sysfs syscall support
    # 这个取决于主板有没有蜂鸣器
    [ ]   Enable PC-Speaker support
```
Processor type and features 下：
```
# EFI 的支持选项：
[*] EFI runtime service support
[*]   EFI stub support
[*]     EFI mixed-mode support

# 虚拟机需要的选项
[*] Linux guest support  --->
    # 半虚拟化
    [*]   Enable paravirtualization code
    # KVM
    [*]   KVM Guest support (including kvmclock)
```
Firmware Drivers 下
```
[*] Export DMI identification via sysfs to userspace
EFI (Extensible Firmware Interface) Support  --->
    < > EFI Variable Support via sysfs
    [*] Export efi runtime maps to sysfs
```
Networking support 下：
```
Networking options  --->
    <*>   The IPv6 protocol --->
```
[*] Enable the block layer 下：
```
 Partition Types  --->
    [*] Advanced partition selection
    [*]   EFI GUID Partition support
```
Device Drivers 下：
```
Generic Driver Options  --->
    [ ] Support for uevent helper
    [*] Maintain a devtmpfs filesystem to mount at /dev
    Firmware loader  --->
        [ ]   Enable the firmware sysfs fallback mechanism
Graphics support  --->
    Frame buffer Devices  --->
        --- Support for frame buffer devices
        [*]   EFI-based Framebuffer Support
    Console display driver support  --->
        [*] Framebuffer Console support
```
File systems 下：
```
[*] Inotify support for userspace
Pseudo filesystems  --->
    [*]   Tmpfs POSIX Access Control Lists
    # 这里可以打成模块也可以直接打进内核
    <M> EFI Variable filesystem
```
其他的都不用管。     
由于 LFS 不使用 initramfs，所以尽量打包进内核，尤其是文件系统相关的不要打成模块。    
## 安装
配置完成，现在开始编译：

    make
安装模块：

    make modules_install
复制内核：

    cp -iv arch/x86_64/boot/bzImage /boot/vmlinuz-5.13.12-lfs-11.0-systemd
此处的内核文件名称可以自行改变，但要以`vmlinuz-`开头。     
复制`System.map`：

    cp -iv System.map /boot/System.map-5.13.12
手册这里将配置`.config`保存到了`/boot`，但我是认为只要不删除源码，放源码树里就行了。     
如果要复制配置的话，运行;

    cp -iv .config /boot/config-5.13.12
安装内核文档：
```
install -d /usr/share/doc/linux-5.13.12
cp -r Documentation/* /usr/share/doc/linux-5.13.12
```
因为不删除源码树要留以后用，而且源码树里可能有不属于`root`的文件，现在要切出目录改变所有者：

    chown -R 0:0 .
## 配置 Linux 内核模块加载顺序
创建文件：
```
install -v -m755 -d /etc/modprobe.d
cat > /etc/modprobe.d/usb.conf << "EOF"
# Begin /etc/modprobe.d/usb.conf

install ohci_hcd /sbin/modprobe ehci_hcd ; /sbin/modprobe -i ohci_hcd ; true
install uhci_hcd /sbin/modprobe ehci_hcd ; /sbin/modprobe -i uhci_hcd ; true

# End /etc/modprobe.d/usb.conf
EOF
```
# 安装 GRUB
## 挂载 EFI 变量文件系统
打开 UEFI 支持的 grub 需要文件系统`efivars`。    
因为需要文件`/sys/firmware/efi/efivars`，而这个文件在非 UEFI 的机器上是不存在的，因此对于我来说需要将 LFS 硬盘迁移到目标机器挂载，现在就需要 live CD 了。    
运行：

    mountpoint /sys/firmware/efi/efivars || mount -v -t efivarfs efivarfs /sys/firmware/efi/efivars
然后安装 grub：

    grub-install --bootloader-id=LFS --recheck
手动写入`grub.cfg`：
```
cat > /boot/grub/grub.cfg << EOF
# Begin /boot/grub/grub.cfg
set default=0
set timeout=5

insmod part_gpt
insmod ext2
set root=(hd0,3)

if loadfont /boot/grub/fonts/unicode.pf2; then
  set gfxmode=auto
  insmod all_video
  terminal_output gfxterm
fi

menuentry "GNU/Linux, Linux 5.13.12-lfs-11.0-systemd"  {
  linux   /boot/vmlinuz-5.13.12-lfs-11.0-systemd root=/dev/sda3 ro
}

menuentry "Firmware Setup" {
  fwsetup
}
EOF
```


# 收尾
现在手册会创建一些描述文件，模板在这，不喜欢可以跳过：
```
echo 11.0-systemd > /etc/lfs-release

cat > /etc/lsb-release << "EOF"
DISTRIB_ID="Linux From Scratch"
DISTRIB_RELEASE="11.0-systemd"
DISTRIB_CODENAME="<your name here>"
DISTRIB_DESCRIPTION="Linux From Scratch"
EOF

cat > /etc/os-release << "EOF"
NAME="Linux From Scratch"
VERSION="11.0-systemd"
ID=lfs
PRETTY_NAME="Linux From Scratch 11.0-systemd"
VERSION_CODENAME="<your name here>"
EOF
```
现在是时候重启了。     
退出 chroot：

    logout
解挂载：

    umount -Rv $LFS
重启

    reboot

至此，LFS 安装完成。