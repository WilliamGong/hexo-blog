---
title: LFS 搭建 2 构建临时系统
date: 2021-11-23T21:34:08+08:00
toc: true
cover:
thumbnail:
categories:
    - LFS
tags:
    - Linux
    - LFS
---
准备工作做完之后，就要开始搭建临时工具链了。     
但说是工具链，其实还包括一些其他工具。

# 开始前的准备
首先当然是登录`lfs`用户。      
之后检查下`$LFS`是否设置正确。

    echo $LFS

之后，需要检查以下部分是否正确设置，虽然大部分都不需要刻意留意，但不排除像我这样有特殊习惯的用户会不满足以下条件。    
1. **shell 用的是 bash。**    
不过目前大部分 shell 都是兼容 bash 的，比如我用的 zsh。
1. **sh 符号链接到 bash。**    
这个绝大多数发行版都满足。
1. **`/usr/bin/awk`符号链接到 gawk。**   
这个也不用在意
1. **`/usr/bin/yacc` 是到 bison 的符号链接，或者是一个执行 bison 的小脚本。**    
嗯，这个也是，除非你用的是 BSD（

此外，对于构建过程，不要把源码放在`/mnt/lfs/tools`里，以免污染。     
在没有特殊说明时，软件包的标准构建步骤如下
1. 解压并 cd 到源码目录
1. 根据 readme 编译，也可以 configure make make install 三连
1. 离开目录，再把它删了。

现在，就正式开始构建。
# 构建交叉编译工具链
现在先把目录切到`$LFS/sources`下。
## Binutils-2.37 - 第一遍
先解压软件包

    tar -xvf binutils-2.37.tar.xz
之后将 patch 放入目录并应用
```
cp binutils-2.37-upstream_fix-1.patch ./binutils-2.37
cd ./binutils-2.37
patch -p1 < binutils-2.37-upstream_fix-1.patch
```
现在需要确认 pty 能否在 chroot 环境里工作

    expect -c "spawn ls"
如果不能显示`spawn ls`的话，就说明环境没有为 pty 正常工作设置好。不过大多数情况下装`expect`包就行了。    
根据 Binutils 手册建议，创建`build`目录：
```
mkdir build
cd build
```
进行 configure
```
../configure --prefix=$LFS/tools \
            --with-sysroot=$LFS \
            --target=$LFS_TGT \
            --disable-nls \
            --disable-werror
```
开始编译：

    make
安装：

    make install -j1
至于加上这个`-j1`的原因，是为了规避 make 中的一个问题。

## GCC 11.2.0  - 第一遍
解压
```
tar -xvf gcc-11.2.0.tar.xz
cd gcc-11.2.0
```
此处 gcc 需要 GMP，MPFR 和 MPC 软件包，将源码解压进 gcc 目录并随 gcc 一起编译
```
tar -xf ../mpfr-4.1.0.tar.xz
mv -v mpfr-4.1.0 mpfr
tar -xf ../gmp-6.2.1.tar.xz
mv -v gmp-6.2.1 gmp
tar -xf ../mpc-1.2.1.tar.gz
mv -v mpc-1.2.1 mpc
```
对于 x86-64 的宿主机，将 64 位库的默认目录设为`lib`
```
case $(uname -m) in
x86_64)
sed -e '/m64=/s/lib64/lib/' \
-i.orig gcc/config/i386/t-linux64
;;
esac
```
设置`build`目录：
```
mkdir build
cd build
```
之后是一段巨长的 configure：
```
../configure \
--target=$LFS_TGT \
--prefix=$LFS/tools \
--with-glibc-version=2.11 \
--with-sysroot=$LFS \
--with-newlib \
--without-headers \
--enable-initfini-array \
--disable-nls \
--disable-shared \
--disable-multilib \
--disable-decimal-float \
--disable-threads \
--disable-libatomic \
--disable-libgomp \
--disable-libquadmath \
--disable-libssp \
--disable-libvtv \
--disable-libstdcxx \
--enable-languages=c,c++
```
毕竟是交叉编译还是第一遍，这里对gcc关闭了一堆功能，主要是避免编译依赖标准库的代码    
编译：

    make
安装：

    make install
此处还要补上`limits.h`，具体原因是在第一次编译时，`$LFS/usr/include/limits.h`还不存在，因此此时 gcc 安装的是不完整的，自给自足的头文件。虽然不加对于编译 glibc 已经足够，但后续步骤需要完整的内部头文件，因此此时手动将 gcc 源码中的相关头文件抽出。
```
cd ..
cat gcc/limitx.h gcc/glimits.h gcc/limity.h > \
`dirname $($LFS_TGT-gcc -print-libgcc-file-name)`/install-tools/include/limits.h
```

## Linux-5.13.12 API 头文件
国际惯例进行解压
```
tar -xvf linux-5.13.12.tar.xz
cd linux-5.13.12
```
这里只需要编译 glibc 用到的头文件，因此在确保目录里没有什么奇奇怪怪的东西（比如从其他地方 copy 进去的配置什么的）

    make mrproper
之后再从源码中提取头文件然后手动复制到工具链目录下
```
make headers
find usr/include -name '.*' -delete
rm usr/include/Makefile
cp -rv usr/include $LFS/usr
```
最后还是一样的删除整个目录。
## Glibc-2.34
glibc 2.34 需要加一个 patch
```
tar -xvf glibc-2.34.tar.xz
cp glibc-2.34-fhs-1.patch glibc-2.34/
cd glibc-2.34/
patch -p1 < glibc-2.34-fhs-1.patch
```
需要创建符合 LSB 的符号链接
```
case $(uname -m) in
i?86) ln -sfv ld-linux.so.2 $LFS/lib/ld-lsb.so.3
;;
x86_64) ln -sfv ../lib/ld-linux-x86-64.so.2 $LFS/lib64
ln -sfv ../lib/ld-linux-x86-64.so.2 $LFS/lib64/ld-lsb-x86-64.so.3
;;
esac
```
之后是创建 build 目录：
```
mkdir -v build
cd build
```
现在要确保`ldconfig`和`sln`被安装到了`/usr/bin`      

    echo "rootsbindir=/usr/sbin" > configparms
configure
```
../configure \
--prefix=/usr \
--host=$LFS_TGT \
--build=$(../scripts/config.guess) \
--enable-kernel=3.2 \
--with-headers=$LFS/usr/include \
libc_cv_slibdir=/usr/lib
```
编译

    make
安装

    make DESTDIR=$LFS install
此处最好检查`$LFS`，如果该变量是空的话 glibc 是会装的宿主机的。     
此处要修复一个`ldd`的硬编码错误：

    sed '/RTLDLIST=/s@/usr@@g' -i $LFS/usr/bin/ldd

## 第一次检查
当上述包都安装完了之后，就需要检查工具链的基本功能能否正常工作。     
找一个不会污染源码的目录，运行
```
echo 'int main(){}' > dummy.c
$LFS_TGT-gcc dummy.c
readelf -l a.out | grep '/ld-linux'
```
如果一切正常，就只会有以下输出
```
[Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```
在 32 位机器上，解释器名字应该是`/lib/ld-linux.so.2.`。    
现在删除刚刚创建的文件

    rm -v dummy.c a.out
现在用 gcc 的工具安装`limits.h`

    $LFS/tools/libexec/gcc/$LFS_TGT/11.2.0/install-tools/mkheaders

> 虽然我在运行时会报错找不到文件，但`limits.h`确实安装到了目录`include-fixed`里
## GCC-11.2.0 里的 Libstdc++ - 第一遍
第一次编译 gcc 时，因为没有 glibc，因此并没有编译 libstdc++，现在安装了 glibc 后就需要编译它了。      
现在解压 gcc 目录并创建 `build`：
```
tar -xvf gcc-11.2.0.tar.xz
cd gcc-11.2.0
mkdir build
cd build
```
之后是 configure
```
../libstdc++-v3/configure \
--host=$LFS_TGT \
--build=$(../config.guess) \
--prefix=/usr \
--disable-multilib \
--disable-nls \
--disable-libstdcxx-pch \
--with-gxx-include-dir=/tools/$LFS_TGT/include/c++/11.2.0
```
之后就是编译安装二连
```
make
make DESTDIR=$LFS install
```

# 构建交叉编译临时工具
现在只是编译安装了基础的工具链，在此之上还要安装一些额外工具。     
虽然说这些工具是在`chroot`之后用的，但现在就要编译。
## M4-1.4.19
```
tar -xvf m4-1.4.19.tar.xz
cd m4-1.4.19
```
此处不需要`build`目录。     
configure
```
./configure --prefix=/usr \
--host=$LFS_TGT \
--build=$(build-aux/config.guess)
```
编译和安装
```
make
make DESTDIR=$LFS install
```
## Ncurses-6.2
```
tar -xvf ncurses-6.2.tar.gz
cd ncurses-6.2
```
首先要确保在 configure 时要找得到`gawk`

    sed -i s/mawk// configure
之后需要编译`tic`程序
```
mkdir build
pushd build
../configure
make -C include
make -C progs tic
popd
```
之后**不要进入`build`**，直接开始 configure
```
./configure --prefix=/usr \
--host=$LFS_TGT \
--build=$(./config.guess) \
--mandir=/usr/share/man \
--with-manpage-format=normal \
--with-shared \
--without-debug \
--without-ada \
--without-normal \
--enable-widec
```
编译

    make
安装
```
make DESTDIR=$LFS TIC_PATH=$(pwd)/build/progs/tic install
echo "INPUT(-lncursesw)" > $LFS/usr/lib/libncurses.so
```
## Bash-5.1.8
```
tar -xvf bash-5.1.8.tar.gz
cd bash-5.1.8
```
configure
```
./configure --prefix=/usr \
--build=$(support/config.guess) \
--host=$LFS_TGT \
--without-bash-malloc
```
编译和安装
```
make
make DESTDIR=$LFS install
```
加上`sh`到`bash`的符号链接

    ln -sv bash $LFS/bin/sh

## Coreutils-8.32
此处要加一个 patch
```
tar -xvf coreutils-8.32.tar.xz
cp coreutils-8.32-i18n-1.patch coreutils-8.32/
cd coreutils-8.32
patch -Np1 -i coreutils-8.32-i18n-1.patch
```
configure
```
./configure --prefix=/usr \
--host=$LFS_TGT \
--build=$(build-aux/config.guess) \
--enable-install-program=hostname \
--enable-no-install-program=kill,uptime
```
编译安装二连
```
make
make DESTDIR=$LFS install
```
此处需要移动部分文件的位置以适应有些硬编码的程序
```
mv -v $LFS/usr/bin/chroot $LFS/usr/sbin
mkdir -pv $LFS/usr/share/man/man8
mv -v $LFS/usr/share/man/man1/chroot.1 $LFS/usr/share/man/man8/chroot.8
sed -i 's/"1"/"8"/' $LFS/usr/share/man/man8/chroot.8
```
## Diffutils-3.8
```
tar -xvf diffutils-3.8.tar.xz
cd diffutils-3.8
```
三连
```
./configure --prefix=/usr --host=$LFS_TGT
make
make DESTDIR=$LFS install
```
## File-5.40
```
tar -xvf file-5.40.tar.gz
cd file-5.40
```
这里的`file`命令需要与宿主机上的版本一致，因此需要特殊编译
```
mkdir build
pushd build
../configure --disable-bzlib \
--disable-libseccomp \
--disable-xzlib \
--disable-zlib
make
popd
```
再进行三连，同样不要进入`build`
```
./configure --prefix=/usr --host=$LFS_TGT --build=$(./config.guess)
make FILE_COMPILE=$(pwd)/build/src/file
make DESTDIR=$LFS install
```
## Findutils-4.8.0
```
tar -xvf findutils-4.8.0.tar.xz
cd findutils-4.8.0
```
configure
```
./configure --prefix=/usr \
--localstatedir=/var/lib/locate \
--host=$LFS_TGT \
--build=$(build-aux/config.guess)
```
编译和安装
```
make
make DESTDIR=$LFS install
```
## Gawk-5.1.0
```
tar -xvf gawk-5.1.0.tar.xz
cd gawk-5.1.0
```
首先确保不安装不需要的文件

    sed -i 's/extras//' Makefile.in
configure
```
./configure --prefix=/usr \
--host=$LFS_TGT \
--build=$(./config.guess)
```
编译和安装
```
make
make DESTDIR=$LFS install
```
## Grep-3.7
```
tar -xvf grep-3.7.tar.xz
cd grep-3.7
```
configure
```
./configure --prefix=/usr \
--host=$LFS_TGT
```
编译和安装
```
make
make DESTDIR=$LFS install
```
## Gzip-1.10
```
tar -xvf gzip-1.10.tar.xz
cd gzip-1.10
./configure --prefix=/usr --host=$LFS_TGT
make
make DESTDIR=$LFS install
```
## Make-4.3
```
tar -xvf make-4.3.tar.gz
cd make-4.3
```
configure
```
./configure --prefix=/usr \
--without-guile \
--host=$LFS_TGT \
--build=$(build-aux/config.guess)
```
编译和安装
```
make
make DESTDIR=$LFS install
```
## Patch-2.7.6
```
tar -xvf patch-2.7.6.tar.xz
cd patch-2.7.6
```
configure
```
./configure --prefix=/usr \
--host=$LFS_TGT \
--build=$(build-aux/config.guess)
```
编译和安装
```
make
make DESTDIR=$LFS install
```
## Sed-4.8
```
tar -xvf sed-4.8.tar.xz
cd sed-4.8
```
configure
```
./configure --prefix=/usr \
--host=$LFS_TGT
```
编译和安装
```
make
make DESTDIR=$LFS install
```
## Tar-1.34
```
tar -xvf tar-1.34.tar.xz
cd tar-1.34
```
configure
```
./configure --prefix=/usr \
--host=$LFS_TGT \
--build=$(build-aux/config.guess)
```
编译和安装
```
make
make DESTDIR=$LFS install
```
## Xz-5.2.5
```
tar -xvf xz-5.2.5.tar.xz
cd xz-5.2.5
```
configure
```
./configure --prefix=/usr \
--host=$LFS_TGT \
--build=$(build-aux/config.guess) \
--disable-static \
--docdir=/usr/share/doc/xz-5.2.5
```
编译和安装
```
make
make DESTDIR=$LFS install
```
## Binutils-2.37 - 第二遍
这一次和第一次差不多，但有一些微妙的差别
```
tar -xvf binutils-2.37.tar.xz
cp binutils-2.37-upstream_fix-1.patch binutils-2.37
cd binutils-2.37
patch -Np1 -i binutils-2.37-upstream_fix-1.patch
mkdir build
cd build
```
configure
```
../configure \
--prefix=/usr \
--build=$(../config.guess) \
--host=$LFS_TGT \
--disable-nls \
--enable-shared \
--disable-werror \
--enable-64-bit-bfd
```
编译

    make
安装，并绕过导致`libctf.so`链接到宿主发行版 zlib 的问题
```
make DESTDIR=$LFS install -j1
install -vm755 libctf/.libs/libctf.so.0.0.0 $LFS/usr/lib
```
## GCC-11.2.0 - 第二遍
```
tar -xvf gcc-11.2.0.tar.xz
cd gcc-11.2.0
```
还是一样的解压 GMP, MPFR 和 MPC
```
tar -xf ../mpfr-4.1.0.tar.xz
mv -v mpfr-4.1.0 mpfr
tar -xf ../gmp-6.2.1.tar.xz
mv -v gmp-6.2.1 gmp
tar -xf ../mpc-1.2.1.tar.gz
mv -v mpc-1.2.1 mpc
```
还是一样的设置 64 位库的符号链接
```
case $(uname -m) in
x86_64)
sed -e '/m64=/s/lib64/lib/' -i.orig gcc/config/i386/t-linux64
;;
esac
```
建立`build`目录
```
mkdir build
cd build
```
创建一个符号链接以在构建`libgcc`时启用对 posix 线程的支持
```
mkdir -pv $LFS_TGT/libgcc
ln -s ../../../libgcc/gthr-posix.h $LFS_TGT/libgcc/gthr-default.h
```
configure
```
../configure \
--build=$(../config.guess) \
--host=$LFS_TGT \
--prefix=/usr \
CC_FOR_TARGET=$LFS_TGT-gcc \
--with-build-sysroot=$LFS \
--enable-initfini-array \
--disable-nls \
--disable-multilib \
--disable-decimal-float \
--disable-libatomic \
--disable-libgomp \
--disable-libquadmath \
--disable-libssp \
--disable-libvtv \
--disable-libstdcxx \
--enable-languages=c,c++
```
编译和安装
```
make
make DESTDIR=$LFS install
```
一样的创建`cc`到`gcc`的符号链接

    ln -sv gcc $LFS/usr/bin/cc

至此本文就到这里，接下来就要进行 chroot 了。
> 11.0 的手册相较于 9.0 而言第二部分的内容实在是太多了……   
不理解为什么要把 chroot 后的操作也要放在这一部分。