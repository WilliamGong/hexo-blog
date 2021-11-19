---
title: LFS 搭建 1 准备工作
date: 2021-11-19T18:53:29+08:00
toc: true
cover:
thumbnail:
categories:
    - LFS
tags:
    - Linux
    - LFS
---
那么，现在就正式开始。      
目前目标机的硬盘在宿主机的位置为`/dev/sdb`。      
# 分区
目前的分区方案如下

|分区位置|大小|类型（挂载点）|
|---|---|---|
|`/dev/sdb1`|256M|EFI 分区|
|`/dev/sdb2`|4G|swap|
|`/dev/sdb3`|剩余部分|`/`|

现在采用 `parted` 进行分区      
```
# parted /dev/sdb
(parted) mklabel gpt
(parted) mkpart ESP fat32 1M 257M
(parted) set 1 boot on
(parted) mkpart primary linux-swap 257M 4353M
(parted) mkpart primary ext4 4353M -1
```
# 建立文件系统
分区完成后，接下来就是格式化。     
目前，ESP 分区采用 fat32，根分区采用 ext4。      
```
mkfs.vfat /dev/sdb1
mkswap /dev/sdb2
mkfs.ext4 /dev/sdb3
```
# 设置环境变量
首先，根据 LFS 手册的要求，设置`LFS`环境变量。    
从现在开始，宿主机就要进入 root 进行操作了，当然理论上一直 sudo 也可以，但为了方便且预防一些奇奇怪怪的错误，还是进入 root 操作吧。    
此处`$LFS`的值为 LFS 根分区在宿主机挂载点的位置。       

    export LFS=/mnt/lfs
当然，也可以直接写在`~/.bashrc`里，方便。     
# 挂载分区
现在就可以挂载分区了。     
再设置好`$LFS`后，进行这些需要输入路径的操作就方便多了。    
考虑到标准位置，把 ESP 挂载到`$LFS/boot/efi`下。     
```
# mount /dev/sdb3 $LFS
# mkdir -p $LFS/boot/efi
# mount /dev/sdb1 $LFS/boot/efi
```
# 准备软件包
首先建立一个软件包存放目录，且这个目录 LFS 会要求打开粘滞模式。
```
mkdir -v $LFS/sources
chmod -v a+wt $LFS/sources
```
这里使用 ustc 的镜像，镜像地址：     
http://mirrors.ustc.edu.cn/lfs/lfs-packages/11.0/    
本来要打算使用镜像里的 `wget-list`，但是好家伙，打开一看地址都是源地址，完美镜像。     
而且此时 LCTT 给的列表是9.0的……      
不过好心的 ustc 给了 tar 包。
```
# cd $LFS/sources
# wget http://mirrors.ustc.edu.cn/lfs/lfs-packages/lfs-packages-11.0.tar
# tar -xvf lfs-packages-11.0.tar
```
此处对文件进行一些整理
```
# mv $LFS/sources/11.0/* $LFS/sources/
# rm -r 11.0/
# rm lfs-packages-11.0.tar
```
此处也可以检查下 checksum。      
# 准备阶段收尾工作
该部分主要是设置目录，用户，环境变量等一系列配置。      
## 建立系统目录
建立一些之后编译安装软件包时会用到的系统目录。     
但此处创建的目录并不完全。       
使用下面的脚本，以 root 运行。     
```
#!/bin/bash

mkdir -pv $LFS/{etc,var} $LFS/usr/{bin,lib,sbin}

for i in bin lib sbin; do
ln -sv usr/$i $LFS/$i
done

case $(uname -m) in
x86_64) mkdir -pv $LFS/lib64 ;;
esac
```
## 建立工具目录
还要建立一个存放临时工具链的目录。
```
# mkdir -pv $LFS/tools
```
> 相比于 9.0 的手册，11.0 版本少了将这个工具目录连接到`/`的操作，emmmmmm

## 创建 LFS 用户
毕竟在 root 下进行编译是十分危险的，创建一个普通用户很有必要。      
虽说自用的用户就行，但在安装过程中还要设置一大堆环境变量，没人想把自己用户的环境变量搞得一团糟吧？      
```
# groupadd lfs
# useradd -s /bin/bash -g lfs -m -k /dev/null lfs
```
此处创建用户的参数就根据自己的喜好了。     
记得设置密码。
```
# passwd lfs
```
接下来要将 `$LFS`的目录的所有权改为 lfs。     
因为指南提供的是多行代码，因此使用脚本运行要方便些。    
```
#!/bin/bash
chown -v lfs $LFS/{usr{,/*},lib,var,etc,bin,sbin,tools}
case $(uname -m) in
x86_64) chown -v lfs $LFS/lib64 ;;
esac
```
> 当然这些目录的所有者后期是要改回来的，不然会出事情的。

同时软件包源码目录的所有者也要改
```
# chown -v lfs $LFS/sources
```
现在，就要登录 lfs 用户进行操作了。     
如果要直接切换，使用`su - lfs`。
## 设置环境变量
首先在`~/.bash_profile`里加上如下内容：
```
exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
```
用于清除多余的环境变量。     
之后再在`~/.bashrc`里加上如下内容：
```
set +h
umask 022
LFS=/mnt/lfs
LC_ALL=POSIX
LFS_TGT=$(uname -m)-lfs-linux-gnu
PATH=/usr/bin
if [ ! -L /bin ]; then PATH=/bin:$PATH; fi
PATH=$LFS/tools/bin:$PATH
CONFIG_SITE=$LFS/usr/share/config.site
export LFS LC_ALL LFS_TGT PATH CONFIG_SITE
```
> 如果 lfs 使用了其他的 shell，需要根据具体 shell 确定写入的文件。    
比如我用的是zsh，以上内容就要写到`~/.zprofile`和`~/.zshrc`里，同时 shell 的路径也要相应改动。    
别无脑写进 bashrc，不然之后环境变量没配置成功还不知道呢。

最后，运行
```
source ~/.bash_profile
```

至此，准备部分结束。