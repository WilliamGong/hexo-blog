---
title: 在 Proxmox VE 中添加 vTPM 的支持
date: 2021-06-25T10:26:26+08:00
toc: true
cover:
thumbnail:
categories:
    - 杂谈
tags:
    - 杂谈
    - Proxmox VE
    - Linux
---

# 前言
虽然 Windows 11 的镜像早就泄露了，但在 6 月 24 日 23 点 Microsoft 才正式发布。发布会结束后，MS 官网放出了 Windows 11 的系统要求，其中一个必须要求是 TPM 2.0。      
然而，Proxmox VE 并没有官方支持 vTPM ， 虽然有qemu 有相应支持，但网上对此的文档很少。因此，本文在此写出让 PVE 支持 qemu  vTPM 的方法以在虚拟机安装 Windows 11 预览版。    
# 关于 Proxmox VE 对 TPM 虚拟化的支持
目前，qemu 已经通过 swtpm 支持了 vTPM ，但Proxmox 对此的支持还在开发中，并且预期时间未知。（Proxmox 工作人员的最近回复在今年 1 月。）     
但可以安装 swtpm 并手动改配置文件，就是有亿点麻烦。      
具体情况见[这里](https://forum.proxmox.com/threads/vtpm-support-do-we-have-guide-to-add-the-vtpm-support.56982/)。      
不过 PVE 对 qemu vTPM 支持的要求已经上 bug 追踪列表了。耐心等吧，总会有的。     
[Bugzilla – Bug 3075](https://bugzilla.proxmox.com/show_bug.cgi?id=3075)     
# 编译与安装 swtpm
## 在 Debian 下编译安装 swtpm
### 下载源码
swtpm 有一个依赖 libtpms 也需要单独编译安装，这里也一起安装了。    
swtpm 的源码

    git clone https://github.com/stefanberger/swtpm.git
libtpms 的源码

### 签出到稳定分支
查看分支

    $ git branch -a
签出时选择最新的稳定版即可。
在写这篇文章时最新版是0.6

    $ git checkout stable-0.6
（这是签出 swtpm 的命令，libtpms 请自行选择版本）
### 编译安装 libtpms
安装依赖:

    sudo apt-get -y install automake autoconf libtool gcc build-essential libssl-dev dh-exec pkg-config gawk
编译:
```
./autogen.sh --with-openssl
mv debian/source debian/source.old
dpkg-buildpackage -us -uc -j4
```
之后回到父目录，就可以看见 libtpms 的 deb 包了。    
安装：

    sudo dpkg -i libtpms0_0*_amd64.deb libtpms-dev_0*_amd64.deb
### 编译 swtpm
安装依赖：

    sudo apt-get -y install  libfuse-dev libglib2.0-dev libgmp-dev expect libtasn1-dev socat tpm-tools python3-twisted gnutls-dev gnutls-bin  libjson-glib-dev python3-setuptools softhsm2 libseccomp-dev
编译：

    dpkg-buildpackage -us -uc -j$(nproc)
### 编译后软件包介绍
编译完成后，编译父目录会生成很多包，其中以下几个包是最终使用的：
+ `libtpms-dev_*_amd64.deb` 这个应该是 swtpm 的编译依赖，但还是装在生产环境上吧，我也不清楚。
+ `libtpms0_*_amd64.deb`   
以上两个包是 libtpms
+ `swtpm-libs_*_amd64.deb`
+ `swtpm_*_amd64.deb`
+ `swtpm-tools_*_amd64.deb`    
以上是 swtpm

其它的包就不用装了，用于 debug 的。
### 福利
已经有人写好一键安装脚本了，在这里：
[rayures/vTPM](https://github.com/rayures/vTPM)   
仅用于 Debian/Ubuntu。
## Gentoo 下的编译
portage 中是有 swtpm 的 ebuild 的，但被`~amd64` mask 了。     
> 因此，生产环境慎用！！！

将 keyword 加入 `/etc/portage/package.keywords`

    =app-crypt/swtpm-0.5.2 ~amd64
    =dev-libs/libtpms-0.8.3 ~amd64

此时运行

    sudo emerge --ask swtpm
即可。
# 在 PVE 中添加 swtpm 设备
首先当然是把编译好的 deb 包装到 PVE 上。    
记得用`dpkg --info`看看依赖，尤其是`swtpm-tools`。   
**在安装`swtpm-tools`时注意先安装它的依赖，不然 dpkg 后再安装依赖的话就会出现循环依赖。**   
swtpm 可以通过套接字/字符设备/CUSE 让 guest 访问 TPM。     
此处使用这个脚本创建套接字设备:
```
#!/bin/bash

i=0
while [ -d /tmp/tpm$i ]; do
let i=i+1
done
tpm=/tmp/tpm$i

mkdir $tpm
echo "Starting $tpm"
sudo swtpm socket  -d --tpmstate dir=$tpm --tpm2 \
             --ctrl type=unixio,path=/$tpm/swtpm-sock &
sleep 2 # this should be changed to a netstat query
```
之后应该可以在`/tmp/`下看见设备。    
## 为虚拟机添加设备
获取 VNC 端口号：
> 我也不知道这是什么，但后面添加参数需要这个数字，求大佬告知。
```
#!/bin/bash
vncport=0
port=5900
while nc -z 127.0.0.1 $port; do
    port=$((port + 1))
    vncport=$((vncport + 1))
done
echo $vncport
```
之后再虚拟机配置文件里加一行：

    args: -drive file=${disk},format=raw,if=virtio,cache=none -chardev socket,id=chrtpm,path=/$tpm/swtpm-sock -tpmdev emulator,id=tpm0,chardev=chrtpm -device tpm-tis,tpmdev=tpm0 -vnc :$nextvnc -m 2048
`${disk}`为虚拟机磁盘镜像路径，`$tpm`为 tpm 设备路径，`$nextvnc`是上面脚本的输出。全部为绝对路径。   
启动虚拟机，就可以再 sealBIOS 里看到TPM 设置了。

## 关于脚本的说明
脚本不是我写的，原帖在这里：    
[S3hh's Blog](https://s3hh.wordpress.com/2018/06/03/tpm-2-0-in-qemu/)    
但该脚本似乎无法直接使用，于是我把它拆成上面的几个脚本和操作。

# 存在的问题
进行上述操作后，如果将虚拟机以裸机启动后再强行关机，会导致无法再启动，需要重新创建设备。