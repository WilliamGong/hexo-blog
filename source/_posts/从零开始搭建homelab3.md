---
title: 从零开始搭建 Home Lab 3 安装 Gentoo 中的那些坑
date: 2021-05-13T14:57:20+08:00
toc: true
cover:
thumbnail:
categories: 
    - Home Lab
tags:
    - Linux
    - Gentoo
---

当 Proxmox VE 已经搭建完成后，就可以准备开发机了。
# 创建虚拟机
其实 PVE 虚拟机创建向导很舒服，在一般情况下照着来就行。但对于 Gentoo，从这里开始就有坑了。    
首先是 CPU，这个虽然不是坑，但这是我的一个小小的建议，将 CPU 类型设置为 host。毕竟由于 Gentoo 的特性，可以针对 CPU 进行优化，对于像好好玩 Gentoo 的人来说，个人认为这一点蛮重要的。而且由于要编译嘛，CPU 性能能榨干一点是一点。    
此外就是各驱动了。**千万不要选 VirtIO 驱动！除非你第一次安装就自己配置内核而且不使用 genkernel 生成的 initramfs。**    
因为以前被坑过无数次了，所以这次第一次安装我选择 genkernel 直接搞定。而 genkernel 在不加参数的情况下是不会选中任何 virtIO 相关选项的……（这是我后来才知道的）。所以如果不想被 genkernel 坑死的话就不要上 virtIO 驱动，至少安装系统时不要选。
# 安装 Gentoo 的指导
本文不会完整记录安装 Gentoo 的流程，因为相比于我自己写的，官方的安装手册要专业得多。而对于大多数流程来说，参考手册就行了。     
此外，相比几年前 Gentoo 安装手册中的不完整而且烂的翻译，现在的中文手册已经看不到英文了，而且中文的翻译质量也不差。所以大可以安心照手册安装，不会有什么让人摸不着头脑的地方的。     
此处为安装手册的链接：[Gentoo AMD64 Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64/zh-cn) 感谢每一位翻译贡献者的努力！    
# 安装 Gentoo 的过程
由于我本次安装使用的 init 是 systemd，而手册默认是为 openRC 准备的，因此我会写下不同于手册的，有关于 systemd 的配置。  
## 选择 stage3
如果要用 systemd 的话，记得选带 systemd 的 stage3。    
虽然使用 openRC 的也行，但是切换完配置后会下载编译 systemd 及其相关依赖，挺耗时间的。    
对了，建议到镜像站下载，比如 tuna。
## 分区
由于 PVE 默认使用 SeaBIOS，所以就不用 ESP 了。但如果硬盘使用 GPT 的话记得加一个 BIOS 启动分区。
## 选择配置文件
如果你使用的是最新的 stage3 而且类型选择正确的话，这一步是可以跳过的。不过保险起见，还是用`eselect profile list`看一眼吧。    
如果你发现系统默认的配置文件不是你想要的话，恭喜你，你多半选错 stage3 了！但其实也没什么，重新选择配置就行了，就是要多等一会了（指至少 1 小时，具体时间取决于机器配置，XD。
## locale 配置
在更改`/etc/locale.gen`，运行`locale-gen`后，最后的选择 locale 就不能按照手册来了。直接修改/创建`/etc/locale.conf`，在里面输入`LANG="en_US.utf8"`即可。
> 此处非常不建议选择有关 zh_CN 的任何 locale。除非在安装时就安装好了桌面环境并确保一旦重新启动就能进入桌面，不然就等着满屏幕的口口口吧。
## 内核配置
如果在这时配置 kernel 也是可以的，但我更喜欢在系统能正常使用的时候再折腾，所以使用 genkernel 一条龙服务吧。    
而如果选择 genkernel 的话，之前的驱动选择就十分重要了。当然，如果你和我一样在安装时选择了 virtIO 驱动，很快就会看到我之前被卡了无数次的错误，以及 genkernel 对于 virtIO 无尽的坑。
## 主机名与 machine ID
systemd 需要一个 machine ID，运行`systemd-machine-id-setup`。    
对于主机名，直接在`/etc/hostname`写就行了。默认该文件是自己创建的，所以看到 nano 显示是新文件时不用惊慌。    
对了，记得把`/etc/hosts`中的`localhost`改为自己的主机名。
## 网络配置
如果使用 dhcp 的话，记得重启后一波`systemctl enable dhcpcd.service`和`systemctl start dhcpcd.service`二连就行。    
但如果你和我一样使用静态 IP 的话，就不能使用 dhcpcd 了。
> 其实按照 wiki，是可以使用 dhcpcd 配置静态 IP 的，但我尝试了没成功。

此处使用 systemd-networkd 配置静态 IP。在`/etc/systemd/network`下创建 network 配置文件，比如下面的配置文件`20-wired.network`：
```
[Match]
Name=enp1s0

[Network]
Address=10.1.10.9/24
Gateway=10.1.10.1
DNS=10.1.10.1
#DNS=8.8.8.8
```
记得把`Name`改为自己的网卡名称。    
对了，重启后也要进行`systemctl enable systemd-networkd.service`与`systemctl start systemd-networkd.service`二连。
## 日志工具
因为 systemd 已经自带了，所以手册关于安装日志工具的部分跳过就好。

# 关于错误：block device is not a valid root device 的解决方法
如果你和我一样在安装时 scsi 控制器选择了 virtIO 驱动时，就会在开机时看到以下类似错误：
```
/dev/loop0: TYPE="squashfs"
/dev/sda2: UUID="eefd6088-354b-4b5b-97d8-5df2df******" TYPE="swap" PARTLABEL="primary" PARTUUID="ea452ed8-8b99-4a26-a662-ab43c******"
............

block device is not a valid root device
```
并且只能进入紧急命令行。      
不要急，进入紧急命令行，看看`/dev/`下有什么。     
如果我没猜错，安装根文件的 sda，要么不见了，要么变成了 hda。      
对于我的情况，是直接不见了。     
这种情况一般是 scsi 控制器出问题了，而且多半是驱动问题，导致根文件所在的硬盘无法加载。    
而我的情况是，内核没有打入任何 virtIO 驱动，initramfs 也没有。
这就是 genkernel 对于 virtIO 的巨坑，因为它对 genkernel 支持不佳，内核编译时不会选中相关选项，就算自己选上了，在制作 initramfs 时也不会打入 virtIO 相关模块。     
如果想要 genkernel 加上 virtIO 选项以及在 initramfs 中打入相关模块，请加上`--virtio`的选项。     
同时，要自己配置内核时，也要选中 virtIO 的相关选项。      
具体可参见[User:Flow/Gentoo as KVM guest](https://wiki.gentoo.org/wiki/User:Flow/Gentoo_as_KVM_guest)