---
title: LFS 搭建 4 正式构建 (1)
date: 2021-12-01T14:30:05+08:00
toc: true
cover:
thumbnail:
categories:
    - LFS
tags:
    - Linux
    - LFS
---

在临时系统构建完成后，从现在开始，就进入正式构建阶段。     
根据 LFS 手册，绝大多数包在编译时都不使用静态库，且一般都会在 configure 中直接关闭，不过 gcc 之类需要的除外。     
在现阶段我并没有移植包管理器的打算，也许以后有时间我会出一篇将 Portage 移植到 LFS 的教程吧。 :D
# 进入 Chroot 环境
每次登录，都要检查`$LFS`：

    echo $LFS
首先先挂载`/dev`和内核虚拟文件系统
```
mount -v --bind /dev $LFS/dev
mount -v --bind /dev/pts $LFS/dev/pts
mount -vt proc proc $LFS/proc
mount -vt sysfs sysfs $LFS/sys
mount -vt tmpfs tmpfs $LFS/run
```
现在开始 chroot
```
chroot "$LFS" /usr/bin/env -i   \
    HOME=/root                  \
    TERM="$TERM"                \
    PS1='(lfs chroot) \u:\w\$ ' \
    PATH=/usr/bin:/usr/sbin \
    /bin/bash --login +h
```

# 构建软件包
## Man-pages-5.13
```
tar -xvf man-pages-5.13.tar.xz
cd man-pages-5.13
make prefix=/usr install
```
## Iana-Etc-20210611
```
tar -xvf iana-etc-20210611.tar.gz
cd iana-etc-20210611
```
这个包只需要将文件复制到`/etc`下即可。

    cp services protocols /etc
## Glibc-2.34
这应该是最后一遍了吧。
```
tar -xvf glibc-2.34.tar.xz
cp glibc-2.34-fhs-1.patch glibc-2.34/
cd glibc-2.34
```
此处需要修复一个安全问题：
```
sed -e '/NOTIFY_REMOVED)/s/)/ \&\& data.attr != NULL)/' \
    -i sysdeps/unix/sysv/linux/mq_notify.c
```
打补丁

    patch -Np1 -i glibc-2.34-fhs-1.patch
> 嗯，手册现在才打 patch，可能是之前编译的都是临时用的所以不用打吧。XD

建立`/build`
```
mkdir build
cd build
```
确保将`ldconfig`和`sln`工具安装到`/usr/sbin`目录中：

    echo "rootsbindir=/usr/sbin" > configparms
configure
```
../configure --prefix=/usr                            \
             --disable-werror                         \
             --enable-kernel=3.2                      \
             --enable-stack-protector=strong          \
             --with-headers=/usr/include              \
             libc_cv_slibdir=/usr/lib
```
编译：

    make

检查：

    make check
glibc 这么大的包，一共 4488 项测试，有几个错误很正常，如果只有几个错误的话忽略就行。此外手册已知会出现两个错误，分别是：
+ `io/tst-lchmod`在 chroot 里会失败
+ `misc/tst-ttyname`在 chroot 里失败

> 我就报了这两个错

安装时，可能会报错`/etc/ld.so.conf`找不到，此错误无害，建立个空的就行

    touch /etc/ld.so.conf
还要修正 Makefile 跳过一个检查

    sed '/test-installation/s@$(PERL)@echo not running@' -i ../Makefile
安装

    make install
改正`ldd`脚本中硬编码的可执行文件加载器路径

    sed '/RTLDLIST=/s@/usr@@g' -i /usr/bin/ldd
安装`nscd`的配置文件和运行时目录：
```
cp -v ../nscd/nscd.conf /etc/nscd.conf
mkdir -pv /var/cache/nscd
```
安装`nscd`对 systemd 的支持
```
install -v -Dm644 ../nscd/nscd.tmpfiles /usr/lib/tmpfiles.d/nscd.conf
install -v -Dm644 ../nscd/nscd.service /usr/lib/systemd/system/nscd.service
```
现在安装 locale。     
手册提供的只能用于满足测试，简体中文是没有的。因此下面的命令我加了`zh_CN-UTF-8`。
> 所以此处会使用安装单个 locale 的指令。
```
mkdir -pv /usr/lib/locale
localedef -i POSIX -f UTF-8 C.UTF-8 2> /dev/null || true
localedef -i cs_CZ -f UTF-8 cs_CZ.UTF-8
localedef -i de_DE -f ISO-8859-1 de_DE
localedef -i de_DE@euro -f ISO-8859-15 de_DE@euro
localedef -i de_DE -f UTF-8 de_DE.UTF-8
localedef -i el_GR -f ISO-8859-7 el_GR
localedef -i en_GB -f ISO-8859-1 en_GB
localedef -i en_GB -f UTF-8 en_GB.UTF-8
localedef -i en_HK -f ISO-8859-1 en_HK
localedef -i en_PH -f ISO-8859-1 en_PH
localedef -i en_US -f ISO-8859-1 en_US
localedef -i en_US -f UTF-8 en_US.UTF-8
localedef -i es_ES -f ISO-8859-15 es_ES@euro
localedef -i es_MX -f ISO-8859-1 es_MX
localedef -i fa_IR -f UTF-8 fa_IR
localedef -i fr_FR -f ISO-8859-1 fr_FR
localedef -i fr_FR@euro -f ISO-8859-15 fr_FR@euro
localedef -i fr_FR -f UTF-8 fr_FR.UTF-8
localedef -i is_IS -f ISO-8859-1 is_IS
localedef -i is_IS -f UTF-8 is_IS.UTF-8
localedef -i it_IT -f ISO-8859-1 it_IT
localedef -i it_IT -f ISO-8859-15 it_IT@euro
localedef -i it_IT -f UTF-8 it_IT.UTF-8
localedef -i ja_JP -f EUC-JP ja_JP
localedef -i ja_JP -f SHIFT_JIS ja_JP.SIJS 2> /dev/null || true
localedef -i ja_JP -f UTF-8 ja_JP.UTF-8
localedef -i nl_NL@euro -f ISO-8859-15 nl_NL@euro
localedef -i ru_RU -f KOI8-R ru_RU.KOI8-R
localedef -i ru_RU -f UTF-8 ru_RU.UTF-8
localedef -i se_NO -f UTF-8 se_NO.UTF-8
localedef -i ta_IN -f UTF-8 ta_IN.UTF-8
localedef -i tr_TR -f UTF-8 tr_TR.UTF-8
localedef -i zh_CN -f GB18030 zh_CN.GB18030
localedef -i zh_HK -f BIG5-HKSCS zh_HK.BIG5-HKSCS
localedef -i zh_TW -f UTF-8 zh_TW.UTF-8
localedef -i zh_CN -f UTF-8 zh_CN.UTF-8
```
当然也可以全部安装，不过需要的时间就很长了：

    make localedata/install-locales
### 配置 Glibc
创建`/etc/nsswitch.conf`：
```
cat > /etc/nsswitch.conf << "EOF"
# Begin /etc/nsswitch.conf

passwd: files
group: files
shadow: files

hosts: files dns
networks: files

protocols: files
services: files
ethers: files
rpc: files

# End /etc/nsswitch.conf
EOF
```
安装时区数据：     
**注意：此时仍在 glibc 的`build`目录下，如果在其他目录要根据路径改下列命令。**
```
tar -xf ../../tzdata2021a.tar.gz

ZONEINFO=/usr/share/zoneinfo
mkdir -pv $ZONEINFO/{posix,right}

for tz in etcetera southamerica northamerica europe africa antarctica  \
          asia australasia backward; do
    zic -L /dev/null   -d $ZONEINFO       ${tz}
    zic -L /dev/null   -d $ZONEINFO/posix ${tz}
    zic -L leapseconds -d $ZONEINFO/right ${tz}
done

cp -v zone.tab zone1970.tab iso3166.tab $ZONEINFO
zic -d $ZONEINFO -p America/New_York
unset ZONEINF
```
此处顺手配置时区：

    ln -sfv /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
现在还需要配置动态加载器。    
创建一个新的`/etc/ld.so.conf`： 
```
cat > /etc/ld.so.conf << "EOF"
# Begin /etc/ld.so.conf
/usr/local/lib
/opt/lib

EOF
```
不过下面的命令可能会更优雅：
```
cat >> /etc/ld.so.conf << "EOF"
# Add an include directory
include /etc/ld.so.conf.d/*.conf

EOF
mkdir -pv /etc/ld.so.conf.d
```
以上两个命令可以全部使用，因为第二个命令不会擦除原`ld.conf.d`的内容
## Zlib-1.2.11
```
tar -xvf zlib-1.2.11.tar.xz
cd zlib-1.2.11
./configure --prefix=/usr
make
make check
make install
```
此处需要删除一些无用的静态库

    rm -fv /usr/lib/libz.a
## Bzip2-1.0.8
这里需要加补丁
```
tar -xvf bzip2-1.0.8.tar.gz
cp bzip2-1.0.8-install_docs-1.patch bzip2-1.0.8
cd bzip2-1.0.8
patch -Np1 -i bzip2-1.0.8-install_docs-1.patch
```
需要修改 Makefile 保证安装的符号链接是相对的。

    sed -i 's@\(ln -s -f \)$(PREFIX)/bin/@\1@' Makefile
确保`man`被安装到正确位置：

    sed -i "s@(PREFIX)/man@(PREFIX)/share/man@g" Makefile
编译前还需要进行一些处理
```
make -f Makefile-libbz2_so
make clean
```
编译：

    make
安装：
```
make PREFIX=/usr install
cp -av libbz2.so.* /usr/lib
ln -sv libbz2.so.1.0.8 /usr/lib/libbz2.so
cp -v bzip2-shared /usr/bin/bzip2
for i in /usr/bin/{bzcat,bunzip2}; do
  ln -sfv bzip2 $i
done
```
此处对其共享库进行了一些处理
删除静态库

    rm -fv /usr/lib/libbz2.a
## Xz-5.2.5
```
tar -xvf xz-5.2.5.tar.xz
cd xz-5.2.5
```
configure
```
./configure --prefix=/usr    \
            --disable-static \
            --docdir=/usr/share/doc/xz-5.2.5
```
编译，检查和安装
```
make
make check
make install
```
## Zstd-1.5.0
```
tar -xvf zstd-1.5.0.tar.gz
cd zstd-1.5.0
make
make check
```
如果编译时出现 "failed" 时，那没有问题，只有 "FAIL"才是失败。但测试需要全部通过。     
安装

    make prefix=/usr install
删除静态库

    rm -v /usr/lib/libzstd.a
## File-5.40
```
tar -xvf file-5.40.tar.gz
cd file-5.40
./configure --prefix=/usr
make
make check
make install
```
## Readline-8.1
```
tar -xvf readline-8.1.tar.gz
cd readline-8.1
```
重新安装`Readline`会导致旧版本的库被重命名为`<库名称>.old`。这一般不是问题，但某些情况下会触发`ldconfig`的一个链接 bug。运行下面的两条 sed 命令防止这种情况： 
```
sed -i '/MV.*old/d' Makefile.in
sed -i '/{OLDSUFF}/c:' support/shlib-install
```
configure
```
./configure --prefix=/usr    \
            --disable-static \
            --with-curses    \
            --docdir=/usr/share/doc/readline-8.1
```
编译和安装：
```
make SHLIB_LIBS="-lncursesw"
make SHLIB_LIBS="-lncursesw" install
```
安装文档（可选）：

    install -v -m644 doc/*.{ps,pdf,html,dvi} /usr/share/doc/readline-8.1
## M4-1.4.19
```
tar -xvf m4-1.4.19.tar.xz
cd m4-1.4.19
./configure --prefix=/usr
make
make check
make install
```
## Bc-5.0.0
```
tar -xvf bc-5.0.0.tar.xz
cd bc-5.0.0
CC=gcc ./configure --prefix=/usr -G -O3
make
make test
make install
```
> 这个`O3`就很灵性

## Flex-2.6.4
```
tar -xvf flex-2.6.4.tar.gz
cd flex-2.6.4
```
configure
```
./configure --prefix=/usr \
            --docdir=/usr/share/doc/flex-2.6.4 \
            --disable-static
```
编译，检查和安装
```
make
make check
make install
```
添加一个`lex`到`flex`的符号链接

    ln -sv flex /usr/bin/lex
## Tcl-8.6.11
这是三个需要用到的测试套件其中之一。    
进入安装目录后，要解压文档，在解压源码时要注意区分。
```
tar -xvf tcl8.6.11-src.tar.gz
cd tcl8.6.11
tar -xf ../tcl8.6.11-html.tar.gz --strip-components=1
```
这里的 configure 有点特殊
```
SRCDIR=$(pwd)
cd unix
./configure --prefix=/usr           \
            --mandir=/usr/share/man \
            $([ "$(uname -m)" = x86_64 ] && echo --enable-64bit)
```
编译，也有点特殊
```
make

sed -e "s|$SRCDIR/unix|/usr/lib|" \
    -e "s|$SRCDIR|/usr/include|"  \
    -i tclConfig.sh

sed -e "s|$SRCDIR/unix/pkgs/tdbc1.1.2|/usr/lib/tdbc1.1.2|" \
    -e "s|$SRCDIR/pkgs/tdbc1.1.2/generic|/usr/include|"    \
    -e "s|$SRCDIR/pkgs/tdbc1.1.2/library|/usr/lib/tcl8.6|" \
    -e "s|$SRCDIR/pkgs/tdbc1.1.2|/usr/include|"            \
    -i pkgs/tdbc1.1.2/tdbcConfig.sh

sed -e "s|$SRCDIR/unix/pkgs/itcl4.2.1|/usr/lib/itcl4.2.1|" \
    -e "s|$SRCDIR/pkgs/itcl4.2.1/generic|/usr/include|"    \
    -e "s|$SRCDIR/pkgs/itcl4.2.1|/usr/include|"            \
    -i pkgs/itcl4.2.1/itclConfig.sh

unset SRCDIR
```
测试

    make test
已知测试`unitInit-1.2`会失败。
> 但我测试时全部成功了

安装，并修改安装好的库的权限：
```
make install
chmod -v u+w /usr/lib/libtcl8.6.so
```
安装头文件，这是等一会要安装的`expect`的依赖

    make install-private-headers
创建符号链接

    ln -sfv tclsh8.6 /usr/bin/tclsh
最后修改一个与 Perl man 冲突的错误：

    mv /usr/share/man/man3/{Thread,Tcl_Thread}.3
## Expect-5.45.4
三个测试套件之二。
```
tar -xvf expect5.45.4.tar.gz
cd expect5.45.4
```
configure
```
./configure --prefix=/usr           \
            --with-tcl=/usr/lib     \
            --enable-shared         \
            --mandir=/usr/share/man \
            --with-tclinclude=/usr/include
```
编译，测试
```
make
make test
```
安装
```
make install
ln -svf expect5.45.4/libexpect5.45.4.so /usr/lib
```
## DejaGNU-1.6.3
三个测试套件之三。     
此处需要`build`
```
tar -xvf dejagnu-1.6.3.tar.gz
cd dejagnu-1.6.3
mkdir -v build
cd       build
```
configure
```
../configure --prefix=/usr
makeinfo --html --no-split -o doc/dejagnu.html ../doc/dejagnu.texi
makeinfo --plaintext       -o doc/dejagnu.txt  ../doc/dejagnu.texi
```
直接安装
```
make install
install -v -dm755  /usr/share/doc/dejagnu-1.6.3
install -v -m644   doc/dejagnu.{html,txt} /usr/share/doc/dejagnu-1.6.3
```
当然也可以测试，虽然没有必要

    make check
## Binutils-2.37
这应该是最后一遍了吧。
```
tar -xvf binutils-2.37.tar.xz
cp binutils-2.37-upstream_fix-1.patch binutils-2.37
cd binutils-2.37
```
同样需要对 pty 进行测试，该命令应返回`spawn ls`：

    expect -c "spawn ls"
手册在这里打补丁，其实在进入目录后就可以打：

    patch -Np1 -i binutils-2.37-upstream_fix-1.patch
绕过关于`man`的一个问题：
```
sed -i '63d' etc/texi2pod.pl
find -name \*.1 -delete
```
创建`build`：
```
mkdir -v build
cd       build
```
configure
```
../configure --prefix=/usr       \
             --enable-gold       \
             --enable-ld=default \
             --enable-plugins    \
             --enable-shared     \
             --disable-werror    \
             --enable-64-bit-bfd \
             --with-system-zlib
```
编译及测试
```
make tooldir=/usr
make -k check
```
**一定要运行测试！**    
已知四项与`zlib`有关的测试会失败。     
> 嗯，我全中。

安装，删除无用的静态库
```
make tooldir=/usr install -j1
rm -fv /usr/lib/lib{bfd,ctf,ctf-nobfd,opcodes}.a
```
## GMP-6.2.1
```
tar -xvf gmp-6.2.1.tar.xz
cd gmp-6.2.1
```
如果宿主机硬件是 64 位但安装的 OS 是 32 位，还有`CFLAGS`变量时，需要运行

    ABI=32 ./configure ...
取消处理器优化，生成通用库。（可选，且如果宿主机 CPU 与目标机一致就不用选，在 live CD 里编译也不用。）
```
cp -v configfsf.guess config.guess
cp -v configfsf.sub   config.sub
```
configure
```
./configure --prefix=/usr    \
            --enable-cxx     \
            --disable-static \
            --docdir=/usr/share/doc/gmp-6.2.1
```
编译并生成文档
```
make
make html
```
测试：     
**一定要测试！**   
因为 GMP 编译时是针对 CPU 高度优化的，有时会错误识别 CPU 功能导致测试时大概率出现一堆`Illegal instruction`。此时就需要重新编译并加上`--build=x86_64-pc-linux-gnu`。     
以下命令查看通过测试的数量，GMP 一共 197 项测试，务必全部通过。

    awk '/# PASS:/{total+=$3} ; END{print total}' gmp-check-log
安装，包括文档
```
make install
make install-html
```
## MPFR-4.1.0
```
tar -xvf mpfr-4.1.0.tar.xz
cd mpfr-4.1.0
```
configure
```
./configure --prefix=/usr        \
            --disable-static     \
            --enable-thread-safe \
            --docdir=/usr/share/doc/mpfr-4.1.0
```
编译并生成文档：
```
make
make html
```
测试：    
**一定要测试！**

    make check
确保全部通过。      
安装
```
make install
make install-html
```
## MPC-1.2.1
```
tar -xvf mpc-1.2.1.tar.gz
cd mpc-1.2.1
```
configure
```
./configure --prefix=/usr    \
            --disable-static \
            --docdir=/usr/share/doc/mpc-1.2.1
```
编译，测试和安装
```
make
make html
make check
make install
make install-html
```
## Attr-2.5.1
```
tar -xvf attr-2.5.1.tar.gz
cd attr-2.5.1
```
configure
```
./configure --prefix=/usr     \
            --disable-static  \
            --sysconfdir=/etc \
            --docdir=/usr/share/doc/attr-2.5.1
```
编译，测试和安装
```
make
make check
make install
```
## Acl-2.3.1
```
tar -xvf acl-2.3.1.tar.xz
cd acl-2.3.1
```
configure
```
./configure --prefix=/usr         \
            --disable-static      \
            --docdir=/usr/share/doc/acl-2.3.1
```
编译和安装：
```
make
make install
```
`acl`的测试依赖于连接了 Acl 的Coreutils，如果要测试，应在 Coreutils 构建完成后进行。    
因此，构建完成后源码目录暂不删除。
## Libcap-2.53
```
tar -xvf libcap-2.53.tar.xz
cd libcap-2.53
```
防止安装静态库：

    sed -i '/install -m.*STA/d' libcap/Makefile
编译，测试，安装
```
make prefix=/usr lib=lib
make test
make prefix=/usr lib=lib install
```
修改权限

    chmod -v 755 /usr/lib/lib{cap,psx}.so.2.53
## Shadow-4.9
```
tar -xvf shadow-4.9.tar.xz
cd shadow-4.9
```
禁止该包安装`groups`程序和它的 man 页面，因为 Coreutils 会提供更好的版本。
```
sed -i 's/groups$(EXEEXT) //' src/Makefile.in
find man -name Makefile.in -exec sed -i 's/groups\.1 / /'   {} \;
find man -name Makefile.in -exec sed -i 's/getspnam\.3 / /' {} \;
find man -name Makefile.in -exec sed -i 's/passwd\.5 / /'   {} \;
```
不使用默认的 crypt 加密方法，使用更安全的 SHA-512 方法加密密码，该方法也允许长度超过 8 个字符的密码。还需要把用户邮箱位置`/var/spool/mail`改为`/var/mail`。另外，从默认的`PATH`中删除`/bin` 和`/sbin`，因为它们只是指向`/usr`中对应目录的符号链接：
```
sed -e 's:#ENCRYPT_METHOD DES:ENCRYPT_METHOD SHA512:' \
    -e 's:/var/spool/mail:/var/mail:'                 \
    -e '/PATH=/{s@/sbin:@@;s@/bin:@@}'                \
    -i etc/login.defs
```
修复程序中的一处低级错误：

    sed -e "224s/rounds/min_rounds/" -i libmisc/salt.c
configure
```
touch /usr/bin/passwd
./configure --sysconfdir=/etc \
            --with-group-name-max-length=32
```
编译：

    make
安装：
```
make exec_prefix=/usr install
make -C man install-man
mkdir -p /etc/default
useradd -D --gid 999
```
### 配置
启用用户密码的加密

    pwconv
启用组密码加密

    grpconv
`shadow`带了一个`useradd`的默认配置文件，其中会为新用户创建邮箱文件，若关闭，运行

    sed -i 's/yes/no/' /etc/default/useradd
### 设置 root 密码

    passwd root
## GCC-11.2.0
这一遍需要巨长的时间。
```
tar -xvf gcc-11.2.0.tar.xz
cd gcc-11.2.0
```
修复在使用 Glibc-2.34 的系统上构建该软件包时导致`libasan.a`无法使用的问题： 
```
sed -e '/static.*SIGSTKSZ/d' \
    -e 's/return kAltStackSize/return SIGSTKSZ * 4/' \
    -i libsanitizer/sanitizer_common/sanitizer_posix_libcdep.cpp
```
修改 64 位库默认路径
```
case $(uname -m) in
  x86_64)
    sed -e '/m64=/s/lib64/lib/' \
        -i.orig gcc/config/i386/t-linux64
  ;;
esac
```
创建`build`
```
mkdir -v build
cd       build
```
configure
```
../configure --prefix=/usr            \
             LD=ld                    \
             --enable-languages=c,c++ \
             --disable-multilib       \
             --disable-bootstrap      \
             --with-system-zlib
```
编译：

    make
增加栈空间

    ulimit -s 32768
以非特权用户身份测试编译结果，但出错时继续执行其他测试： 
```
chown -Rv tester . 
su tester -c "PATH=$PATH make -k check"
```
查看摘要

    ../contrib/test_summary
已知错误如下：
+ 已知八项与`analyzer`相关的测试会失败。     
+ 已知一项名为`asan_test.C`的测试会失败。    
+ 在`libstdc++`中，已知一项名为`49745.cc`的测试由于 Glibc 中头文件依赖关系的变化而失败。    
+ 在`libstdc++`测试中，一项`numpuct`测试和六项与 get_time 有关的测试会失败。这是由于 glibc 更新了 locale 定义，但是 libstdc++ 尚不支持这些变化。    

同时，还有少量错误是十分正常的，毕竟这是 gcc。 （    
**哪怕测试时间比编译时间还要长十倍也要坚持测试！**
> 但时间真的太长了……我跑了近 5 个小时，然而编译只需要 30 分钟不到

安装该软件包，并移除一个不需要的目录： 
```
make install
rm -rf /usr/lib/gcc/$(gcc -dumpmachine)/11.2.0/include-fixed/bits/
```
修正之前因为测试而临时更改的文件所有者
```
chown -v -R root:root \
    /usr/lib/gcc/*linux-gnu/11.2.0/include{,-fixed}
```
创建一个符号链接
```
ln -svr /usr/bin/cpp /usr/lib
ln -sfv ../../libexec/gcc/$(gcc -dumpmachine)/11.2.0/liblto_plugin.so \
        /usr/lib/bfd-plugins/
```
## 第二次检查
现在工具链已经完成安装，需要进行一次完整的确认。     
离开源码目录，然后
```
echo 'int main(){}' > dummy.c
cc dummy.c -v -Wl,--verbose &> dummy.log
readelf -l a.out | grep ': /lib'
```
如果正常，输出的最后一行会是（具体名称取决于平台）

    [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
然后

    grep -o '/usr/lib.*/crt[1in].*succeeded' dummy.log
正常输出应该是这样：
```
/usr/lib/gcc/x86_64-pc-linux-gnu/11.2.0/../../../../lib/crt1.o succeeded
/usr/lib/gcc/x86_64-pc-linux-gnu/11.2.0/../../../../lib/crti.o succeeded
/usr/lib/gcc/x86_64-pc-linux-gnu/11.2.0/../../../../lib/crtn.o succeeded
```
`gcc`应该找到所有三个`crt*.o`文件，它们应该位于`/usr/lib`目录中。     
确认编译器能正确查找头文件： 

    grep -B4 '^ /usr/include' dummy.log
这是正常输出：
```
#include <...> search starts here:
 /usr/lib/gcc/x86_64-pc-linux-gnu/11.2.0/include
 /usr/local/include
 /usr/lib/gcc/x86_64-pc-linux-gnu/11.2.0/include-fixed
 /usr/include
```
其中三元组的名称取决于平台。     
确认新的链接器使用了正确的搜索路径： 

    grep 'SEARCH.*/usr/lib' dummy.log |sed 's|; |\n|g'
路径应该要包含
```
SEARCH_DIR("/usr/x86_64-pc-linux-gnu/lib64")
SEARCH_DIR("/usr/local/lib64")
SEARCH_DIR("/lib64")
SEARCH_DIR("/usr/lib64")
SEARCH_DIR("/usr/x86_64-pc-linux-gnu/lib")
SEARCH_DIR("/usr/local/lib")
SEARCH_DIR("/lib")
SEARCH_DIR("/usr/lib");
```
当然 32 位的路径会有不同。     
确认使用了正确的 libc： 

    grep "/lib.*/libc.so.6 " dummy.log
应该输出

    attempt to open /usr/lib/libc.so.6 succeeded
确认 GCC 使用了正确的动态链接器： 

    grep found dummy.log
正常输出：

    found ld-linux-x86-64.so.2 at /usr/lib/ld-linux-x86-64.so.2
如果有问题，一定要修复，不能硬着头皮往下做，不然更折腾人。    
如果一切正常，删除测试文件：

    rm -v dummy.c a.out dummy.log
最后移动一个位置不正确的文件： 
```
mkdir -pv /usr/share/gdb/auto-load/usr/lib
mv -v /usr/lib/*gdb.py /usr/share/gdb/auto-load/usr/lib
```
## Pkg-config-0.29.2
```
tar -xvf pkg-config-0.29.2.tar.gz
cd pkg-config-0.29.2
```
configure
```
./configure --prefix=/usr              \
            --with-internal-glib       \
            --disable-host-tool        \
            --docdir=/usr/share/doc/pkg-config-0.29.2
```
编译，检查，安装
```
make
make check
make install
```
## Ncurses-6.2
```
tar -xvf ncurses-6.2.tar.gz
cd ncurses-6.2
```
configure
```
./configure --prefix=/usr           \
            --mandir=/usr/share/man \
            --with-shared           \
            --without-debug         \
            --without-normal        \
            --enable-pc-files       \
            --enable-widec
```
编译，安装
```
make
make install
```
这个包有测试，但需要安装后面的包，所以跳过。     
创建符号链接：
```
for lib in ncurses form panel menu ; do
    rm -vf                    /usr/lib/lib${lib}.so
    echo "INPUT(-l${lib}w)" > /usr/lib/lib${lib}.so
    ln -sfv ${lib}w.pc        /usr/lib/pkgconfig/${lib}.pc
done

rm -vf                     /usr/lib/libcursesw.so
echo "INPUT(-lncursesw)" > /usr/lib/libcursesw.so
ln -sfv libncurses.so      /usr/lib/libcurses.so
```
删除一个 configure 脚本未处理的静态库： 

    rm -fv /usr/lib/libncurses++w.a
安装文档（可选）：
```
mkdir -v       /usr/share/doc/ncurses-6.2
cp -v -R doc/* /usr/share/doc/ncurses-6.2
```
> 注意：此处没有创建非宽字符的库，部分二进制包会依赖于它。

## Sed-4.8
```
tar -xvf sed-4.8.tar.xz
cd sed-4.8
./configure --prefix=/usr
make
make html
```
测试：
```
chown -Rv tester .
su tester -c "PATH=$PATH make check"
```
> 我在检查时会报错：`inplace-selinux.sh: set-up failure: CONFIG_HEADER not defined`，看上去与 selinux 有关，不用 selinux 应该不受影响。     
但具体原因未知。      

安装：
```
make install
install -d -m755           /usr/share/doc/sed-4.8
install -m644 doc/sed.html /usr/share/doc/sed-4.8
```
## Psmisc-23.4
```
tar -xvf psmisc-23.4.tar.xz
cd psmisc-23.4
./configure --prefix=/usr
make
make install
```
## Gettext-0.21
```
tar -xvf gettext-0.21.tar.xz
cd gettext-0.21
```
configure
```
./configure --prefix=/usr    \
            --disable-static \
            --docdir=/usr/share/doc/gettext-0.21
```
编译，检查
```
make
make check
```
测试时间会和编译时间一样长。      
安装：
```
make install
chmod -v 0755 /usr/lib/preloadable_libintl.so
```
## Bison-3.7.6
```
tar -xvf bison-3.7.6.tar.xz
cd bison-3.7.6
./configure --prefix=/usr --docdir=/usr/share/doc/bison-3.7.6
make
make install
```
可以测试，不过时间有点长。（可选）：

    make check
## Grep-3.7
```
tar -xvf grep-3.7.tar.xz
cd grep-3.7
./configure --prefix=/usr
make
make check
make install
```
## Bash-5.1.8
```
tar -xvf bash-5.1.8.tar.gz
cd bash-5.1.8
```
configure
```
./configure --prefix=/usr                      \
            --docdir=/usr/share/doc/bash-5.1.8 \
            --without-bash-malloc              \
            --with-installed-readline
```
编译：

    make
测试（可选）：
```
chown -Rv tester .

su -s /usr/bin/expect tester << EOF
set timeout -1
spawn make tests
expect eof
lassign [wait] _ _ _ value
exit $value
EOF
```
安装：

    make install
换 shell：

    exec /bin/bash --login +h
## Libtool-2.4.6
```
tar -xvf libtool-2.4.6.tar.xz
cd libtool-2.4.6
./configure --prefix=/usr
make
```
测试，这里建议打开多线程：

    make check TESTSUITEFLAGS=-j5
这里的线程数一般为逻辑 CPU 数 + 1。     
已知会有5个因循环依赖导致的失败，但安装 automake 后可通过。     
安装，删除静态库：
```
make install
rm -fv /usr/lib/libltdl.a
```
## GDBM-1.20
```
tar -xvf gdbm-1.20.tar.gz
cd gdbm-1.20
```
configure
```
./configure --prefix=/usr    \
            --disable-static \
            --enable-libgdbm-compat
```
编译，测试：
```
make
make -k check
```
已知`gdbmtool`测试会失败。      
安装：

    make install
## Gperf-3.1
```
tar -xvf gperf-3.1.tar.gz
cd gperf-3.1
./configure --prefix=/usr --docdir=/usr/share/doc/gperf-3.1
make -j1 check
make install
```
这里开`-j1`是因为多线程会导致错误。      
## Expat-2.4.1
```
tar -xvf expat-2.4.1.tar.xz
cd expat-2.4.1
```
configure
```
./configure --prefix=/usr    \
            --disable-static \
            --docdir=/usr/share/doc/expat-2.4.1
```
编译，检查，安装：
```
make
make check
make install
```
安装文档（可选）：

    install -v -m644 doc/*.{html,png,css} /usr/share/doc/expat-2.4.1

至此，软件包安装完成过半。      
剩下的下篇再继续。
> 76 个包，属实是体力活……