---
title: LFS 搭建3 Chroot 以及构建额外工具
date: 2021-11-28T15:15:48+08:00
toc: true
cover:
thumbnail:
categories:
    - LFS
tags:
    - Linux
    - LFS
---

现在开始就要 chroot 了，同时现在开始用户也要从`lfs`到`root`。     
不过这个顺序相比 9.0 差别太大了吧……   
那么，现在就`su`吧。   
记得设置`$LFS`:

    export LFS=/mnt/lfs
# 改变 LFS 系统目录的所有者
毕竟一个正常的系统其系统文件所有者不可能是`lfs`
```
chown -R root:root $LFS/{usr,lib,var,etc,bin,sbin,tools}
case $(uname -m) in
x86_64) chown -R root:root $LFS/lib64 ;;
esac
```
# 准备虚拟内核文件系统
首先需要建立 dev,proc,sys,run 目录

    mkdir -pv $LFS/{dev,proc,sys,run}
## 创建初始设备节点
主要是`null`和`console`
```
mknod -m 600 $LFS/dev/console c 5 1
mknod -m 666 $LFS/dev/null c 1 3
```
## 挂载 /dev

    mount -v --bind /dev $LFS/dev
## 挂载虚拟内核文件系统
```
mount -v --bind /dev/pts $LFS/dev/pts
mount -vt proc proc $LFS/proc
mount -vt sysfs sysfs $LFS/sys
mount -vt tmpfs tmpfs $LFS/run
```
此外，还要创建一些目录
```
if [ -h $LFS/dev/shm ]; then
  mkdir -pv $LFS/$(readlink $LFS/dev/shm)
fi
```
# Chroot
```
chroot "$LFS" /usr/bin/env -i   \
    HOME=/root                  \
    TERM="$TERM"                \
    PS1='(lfs chroot) \u:\w\$ ' \
    PATH=/usr/bin:/usr/sbin \
    /bin/bash --login +h
```
这时的 bash 会显示 I have no name，因为没有`/etc/passwd`，这是正常的，虽然有点丑 :D
## 退出时进入 Chroot 的操作
如果只是退出而不关机/重启的话，直接运行上面的命令就行，不然就要重新挂载这些文件系统。    
不过建议每次 chroot 前都检查一次。
# 创建目录
现在再创建一些还需要的目录

    mkdir -pv /{boot,home,mnt,opt,srv}
其实有一些目录已经创建了，但这里为了防止遗漏手册还是加上了这些目录。    
现在创建子目录
```
mkdir -pv /etc/{opt,sysconfig}
mkdir -pv /lib/firmware
mkdir -pv /media/{floppy,cdrom}
mkdir -pv /usr/{,local/}{include,src}
mkdir -pv /usr/local/{bin,lib,sbin}
mkdir -pv /usr/{,local/}share/{color,dict,doc,info,locale,man}
mkdir -pv /usr/{,local/}share/{misc,terminfo,zoneinfo}
mkdir -pv /usr/{,local/}share/man/man{1..8}
mkdir -pv /var/{cache,local,log,mail,opt,spool}
mkdir -pv /var/lib/{color,misc,locate}
ln -sfv /run /var/run
ln -sfv /run/lock /var/lock
install -dv -m 0750 /root
install -dv -m 1777 /tmp /var/tmp
```
以上目录结构全部基于 FHS，但只加了必要的目录。
# 创建必要的文件和符号链接
先加上从`/etc/mtab`到`/proc`的符号链接

    ln -sv /proc/self/mounts /etc/mtab
然后是创建一个最简单的`/etc/hosts`
```
cat > /etc/hosts << EOF
127.0.0.1  localhost $(hostname)
::1        localhost
EOF
```
> 其实这里的`$(hostname)`在其他发行版里是需要手动改的

创建`/etc/passwd`：
```
cat > /etc/passwd << "EOF"
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/dev/null:/bin/false
daemon:x:6:6:Daemon User:/dev/null:/bin/false
messagebus:x:18:18:D-Bus Message Daemon User:/run/dbus:/bin/false
systemd-bus-proxy:x:72:72:systemd Bus Proxy:/:/bin/false
systemd-journal-gateway:x:73:73:systemd Journal Gateway:/:/bin/false
systemd-journal-remote:x:74:74:systemd Journal Remote:/:/bin/false
systemd-journal-upload:x:75:75:systemd Journal Upload:/:/bin/false
systemd-network:x:76:76:systemd Network Management:/:/bin/false
systemd-resolve:x:77:77:systemd Resolver:/:/bin/false
systemd-timesync:x:78:78:systemd Time Synchronization:/:/bin/false
systemd-coredump:x:79:79:systemd Core Dumper:/:/bin/false
uuidd:x:80:80:UUID Generation Daemon User:/dev/null:/bin/false
systemd-oom:x:81:81:systemd Out Of Memory Daemon:/:/bin/false
nobody:x:99:99:Unprivileged User:/dev/null:/bin/false
EOF
```
可以看出这里加上了 systemd 需要的用户。     
创建`/etc/group`：
```
cat > /etc/group << "EOF"
root:x:0:
bin:x:1:daemon
sys:x:2:
kmem:x:3:
tape:x:4:
tty:x:5:
daemon:x:6:
floppy:x:7:
disk:x:8:
lp:x:9:
dialout:x:10:
audio:x:11:
video:x:12:
utmp:x:13:
usb:x:14:
cdrom:x:15:
adm:x:16:
messagebus:x:18:
systemd-journal:x:23:
input:x:24:
mail:x:34:
kvm:x:61:
systemd-bus-proxy:x:72:
systemd-journal-gateway:x:73:
systemd-journal-remote:x:74:
systemd-journal-upload:x:75:
systemd-network:x:76:
systemd-resolve:x:77:
systemd-timesync:x:78:
systemd-coredump:x:79:
uuidd:x:80:
systemd-oom:x:81:81:
wheel:x:97:
nogroup:x:99:
users:x:999:
EOF
```
这里用户组的标准可以看作 LFS 自己的标准了，其中用户组可以根据需要自行修改。    
现在需要创建一个之后的测试中需要用到的用户，不过是临时的，最后会删除。
```
echo "tester:x:101:101::/home/tester:/bin/bash" >> /etc/passwd
echo "tester:x:101:" >> /etc/group
install -o tester -d /home/tester
```
现在，赋予 bash 以名字（无故中二 XD）：

    exec /bin/bash --login +h
还要创建一些日志文件，虽然都是空的：
```
touch /var/log/{btmp,lastlog,faillog,wtmp}
chgrp -v utmp /var/log/lastlog
chmod -v 664  /var/log/lastlog
chmod -v 600  /var/log/btmp
```
至此，初步配置完成，现在该安装额外工具了。

# 构建额外工具
你没理解错，以下部分全部都以`root`用户编译。
## GCC-11.2.0 中的 Libstdc++ - 第二遍
在前面第二遍编译 gcc 时，并没有编译它，因为它不能用宿主机工具链编译。    
```
tar -xvf gcc-11.2.0.tar.xz
cd gcc-11.2.0
```
创建一个符号链接

    ln -s gthr-posix.h libgcc/gthr-default.h
建立`build`目录
```
mkdir build
cd build
```
configure
```
../libstdc++-v3/configure \
CXXFLAGS="-g -O2 -D_GNU_SOURCE" \
--prefix=/usr \
--disable-multilib \
--disable-nls \
--host=$(uname -m)-lfs-linux-gnu \
--disable-libstdcxx-pch
```
编译和安装
```
make
make install
```
## Gettext-0.21
这里只安装`msgfmt`，`msgmerge`，以及`xgettext`这三个程序
```
tar -xvf gettext-0.21.tar.xz
cd gettext-0.21
./configure --disable-shared
make
cp -v gettext-tools/src/{msgfmt,msgmerge,xgettext} /usr/bin
```
## Bison-3.7.6
```
tar -xvf bison-3.7.6.tar.xz
cd bison-3.7.6
```
configure
```
./configure --prefix=/usr \
--docdir=/usr/share/doc/bison-3.7.6
```
编译和安装
```
make
make install
```
## Perl-5.34.0
此处要加一个 patch
```
tar -xvf perl-5.34.0.tar.xz
cp perl-5.34.0-upstream_fixes-1.patch perl-5.34.0/
cd perl-5.34.0
patch -Np1 -i perl-5.34.0-upstream_fixes-1.patch
```
configure
```
sh Configure -des \
-Dprefix=/usr \
-Dvendorprefix=/usr \
-Dprivlib=/usr/lib/perl5/5.34/core_perl \
-Darchlib=/usr/lib/perl5/5.34/core_perl \
-Dsitelib=/usr/lib/perl5/5.34/site_perl \
-Dsitearch=/usr/lib/perl5/5.34/site_perl \
-Dvendorlib=/usr/lib/perl5/5.34/vendor_perl \
-Dvendorarch=/usr/lib/perl5/5.34/vendor_perl
```
编译和安装
```
make
make install
```
## Python-3.9.6
这里有两个 python 包，`Python-3.9.6.tar.xz`和`python-3.9.6-docs-html.tar.bz2`。    
解压前面那个，后面的那个包因为没安装`bzip2`无法解压。    
注意区分大小写。
```
tar -xvf Python-3.9.6.tar.xz
cd Python-3.9.6
```
configure
```
./configure --prefix=/usr \
--enable-shared \
--without-ensurepip
```
编译和安装
```
make
make install
```
这里有部分模块是无法编译的，但 make 还是会报错（还是 fatal error），这里只要最外面的 make 执行成功即可。
## Texinfo-6.8
```
tar -xvf texinfo-6.8.tar.xz
cd texinfo-6.8
```
在编译之前，需要修复在使用 Glibc-2.34 或更新版本时编译会出现的问题
```
sed -e 's/__attribute_nonnull__/__nonnull/' \
    -i gnulib/lib/malloc/dynarray-skeleton.c
```
之后
```
./configure --prefix=/usr
make
make install
```
## Util-linux-2.37.2
```
tar -xvf util-linux-2.37.2.tar.xz
cd util-linux-2.37.2
```
根据 FHS 的建议，使用`/var/lib/hwclock`：

    mkdir -pv /var/lib/hwclock
configure
```
./configure ADJTIME_PATH=/var/lib/hwclock/adjtime \
--libdir=/usr/lib \
--docdir=/usr/share/doc/util-linux-2.37.2 \
--disable-chfn-chsh \
--disable-login \
--disable-nologin \
--disable-su \
--disable-setpriv \
--disable-runuser \
--disable-pylibmount \
--disable-static \
--without-python \
runstatedir=/run
```
编译和安装
```
make
make install
```
至此，额外的工具就构建完成了。

# 清理
首先清理文档

    rm -rf /usr/share/{info,man,doc}/*
之后是`libtool`的`*.la`文件

    find /usr/{lib,libexec} -name \*.la -delete
最后删除`/tools`，现在已经不需要了

    rm -rf /tools

# 备份
这是可选的，但小心一点总没错。    
以下操作在 chroot 外进行，且使用`root`用户。    

    exit
备份需要至少 1G 空间。     
先解除内核虚拟文件系统的挂载：
```
umount $LFS/dev{/pts,}
umount $LFS/{sys,proc,run}
```
按照手册默认路径，备份于`/root`
```
cd $LFS 
tar -cJpf $HOME/lfs-temp-tools-11.0-systemd.tar.xz .
```
# 还原
需要还原时，运行
```
cd $LFS
rm -rf ./*
tar -xpf $HOME/lfs-temp-tools-11.0-systemd.tar.xz
```
**运行前务必检查`$LFS`是否正确设置！**
> 不然你宿主系统就 gg 了，`rm -rf /*`了解一下。

至此，构建临时系统部分结束。