---
title: 从零开始搭建 Home Lab 0 硬件的选择与架构方案的确定
date: 2021-03-29T14:06:20+08:00
toc: true
cover:
thumbnail:
categories:
    - Home Lab
tags: 
    - 硬件
    - 网络
    - Linux
---

# 前言
搭建 Home Lab 的想法，起源于我有一次运行虚拟机的时候。    
虽然我现在的主力笔记本性能不差，16G 内存 + 256G SSD，但众所周知 Chrome 是著名的性能消耗大户，导致我的内存有一半被它吞了；而且由于笔记本上安装的重型软件和游戏实在太多了，尤其是游戏，占了 60G+ 的空间，导致剩余硬盘空间捉襟见肘。因此每一次使用虚拟机时，都要扣扣索索的盘算着分配的内存和虚拟磁盘的容量。更令人恼火的是，每次创建虚拟机，都要删掉一个游戏或者是占用了大量磁盘空间的软件，而且每次跑虚拟机只是测试一下而已，没过多久就要删，而删去虚拟机的时候再去下载这些程序是十分痛苦的事情。因此我便产生了专门买服务器跑虚拟机的想法。     
而搭建 Home Lab 的另外一个原因，是因为折腾。     
曾经我一直在一台 10 年前购入的 Lenovo 笔记本上跑 Gentoo，但由于**各 种 各 样**的原因，Gentoo 一直没安装成功，这称为了我心中的一根刺；而且我手头缺少一台 Linux 开发机，虽然 WSL 已经能满足大部分需求，但有些东西是 WSL 做不到的。而搭建一台 Home Lab，能做到All in One，开发机什么的将不再是问题。      
> 虽然缺少 Linux 开发机是一个伪需求……

# 对于Home Lab的需求
既然要选择买服务器搭建 Home Lab，那就一步到位吧。        
那我对 Home Lab的需求是什么呢？
1. **NAS**。我喜欢屯资源，什么无损音乐，番剧电影，盗版游戏加起来快有几百 G 了，虽然手头有个 2T 的移动硬盘，但每一次连接和卸载移动硬盘十分麻烦；而且我经常挂机下载，一挂机就是几天，对于笔记本而言总会有稳定性问题，用专用的机器进行显然要好很多。
1. **开发机**。毕竟我要折腾 Gentoo，天天跑编译，CPU 性能还是挺重要的，至少成品 NAS 常用的 Atom，赛扬会有些吃力。
1. **测试机**。有时我需要一台 Windows 测试机试毒养蛊，有时又会开一台 Linux 虚拟机折腾。不过由于只是测试，性能需求会小很多。

# 硬件的选择
本人学生党，对硬件的选择自然是越便宜越好，但在金钱与性能之间权衡是一件很痛苦的事情。死来想去，确认了如下方案。
## 服务器主机
看了几个月，最终确认了购入服务器的型号：HPE ProLiant MicroServer Gen10 Plus。      
为什么要选择这台机器呢？
1. **小**。机身设计十分紧凑，应该是四盘位微型服务器的极限了，而且高度只有它的前代产品的一半左右。放在宿舍里也不占空间，也容易搬。
1. **性能不差**。高配的版本使用的 E3-2200 CPU 性能肯定不辍，但多达 71W 的 TDP 和价钱使我果断放弃，但低配的 G5420 性能也不差。
1. **易于拆装和升级**。不像一部分　NAS　将　CPU　焊在主板上，这台机器　CPU　可以拆卸，而且接口是　LGA1151，虽然不能换大部分至强 CPU，但可以装大部分桌面级 CPU 和一部分 E3，并且高性能服务器级 CPU 与我无缘。虽然我不会上桌面级的 U，但至少还有的选。内存标准最大容量 32G，而且实测最高可以到 64G（当然也与选择的 CPU 有关）。两个标准的 DDR4 内存插槽，支持双通道和 ECC。此外还配有一个标准的 PCI-E 3.0 插槽和一个专用于 ILO 的阉割版 PCI-E 插槽（等于除了插 ILO 网卡外没什么用）。不过这也是它的一个缺点。（但至少还有一个插槽不是吗？）

我的机器是在闲鱼上买的二手未开封机器，相比与狗东上 6000+ 的价格，闲鱼上的价格只有三分之二，而且和全新的没区别。（除了容易被坑，毕竟虽然只有三分之二还是 将近 4000 RMB，被骗了就真的难受了。）
## 内存与硬盘
内存本来是准备一条 16G 的，但下单的时候没注意买成了两条，于是变成了 32G。型号的光威奕 PRO，国产颗粒。虽说支持国产是一个因素，但更重要的原因是便宜。     
至于硬盘，由于要建 NAS，硬盘不敢买差的，于是选择了 2T 的希捷酷狼和一个金士顿的 128G SATA SSD 用作系统盘。
## 网络设备
由于要开各种网络服务，一个路由器是很有必要的，但只需要有就行了。不过由于需要校园网拨号，于是买了一个二手小娱 C3 刷 Open WRT，不到 100 RMB。
# 系统架构的确定
曾经我打算在主机上直接装 Gentoo，其他的开虚拟机，但毕竟实验/开发环境天天挂，最后还是选择在主机上装 Hypervisor，各种服务跑虚拟机上的方案。这个 Hypervisor 将同时运行至少两台虚拟机，NAS 和开发机。有时还要同时运行测试机和跑各种 docker/LXC 的虚拟机。    
Hypervisor 我最终选择了 Proxmox VE。这玩意如果不订阅每次登录都要弹警告很烦人，因此我曾一度打算装 kvm 和 qemu 自己糊，但实际用起来后我直呼真香。NAS 我选择了OpenMediaVault，开发机不用多说自然是 Gentoo。