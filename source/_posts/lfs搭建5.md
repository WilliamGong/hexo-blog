---
title: LFS 搭建 5 正式构建 (2)
date: 2021-12-02T21:47:11+08:00
toc: true
cover:
thumbnail:
categories:
    - LFS
tags:
    - Linux
    - LFS
---

现在继续构建剩下的软件包。
# 构建软件包
## Inetutils-2.1
```
tar -xvf inetutils-2.1.tar.xz
cd inetutils-2.1
```
configure
```
./configure --prefix=/usr        \
            --bindir=/usr/bin    \
            --localstatedir=/var \
            --disable-logger     \
            --disable-whois      \
            --disable-rcp        \
            --disable-rexec      \
            --disable-rlogin     \
            --disable-rsh        \
            --disable-servers
```
编译，检查，安装：
```
make
make check
make install
```
移动位置：

    mv -v /usr/{,s}bin/ifconfig
## Less-590
```
tar -xvf less-590.tar.gz
cd less-590
./configure --prefix=/usr --sysconfdir=/etc
make
make install
```
## Perl-5.34.0
```
tar -xvf perl-5.34.0.tar.xz
cp perl-5.34.0-upstream_fixes-1.patch perl-5.34.0
cd perl-5.34.0
patch -Np1 -i perl-5.34.0-upstream_fixes-1.patch
```
让`perl`找到已经安装好的库：
```
export BUILD_ZLIB=False
export BUILD_BZIP2=0
```
configure
```
sh Configure -des                                         \
             -Dprefix=/usr                                \
             -Dvendorprefix=/usr                          \
             -Dprivlib=/usr/lib/perl5/5.34/core_perl      \
             -Darchlib=/usr/lib/perl5/5.34/core_perl      \
             -Dsitelib=/usr/lib/perl5/5.34/site_perl      \
             -Dsitearch=/usr/lib/perl5/5.34/site_perl     \
             -Dvendorlib=/usr/lib/perl5/5.34/vendor_perl  \
             -Dvendorarch=/usr/lib/perl5/5.34/vendor_perl \
             -Dman1dir=/usr/share/man/man1                \
             -Dman3dir=/usr/share/man/man3                \
             -Dpager="/usr/bin/less -isR"                 \
             -Duseshrplib                                 \
             -Dusethreads
```
编译，测试，安装，清理环境变量：
```
make
make test
make install
unset BUILD_ZLIB BUILD_BZIP2
```
## XML::Parser-2.46
```
tar -xvf XML-Parser-2.46.tar.gz
cd XML-Parser-2.46
perl Makefile.PL
make
make test
make install
```
## Intltool-0.51.0
```
tar -xvf intltool-0.51.0.tar.gz
cd intltool-0.51.0
```
修复由`perl-5.22`及更新版本导致的警告：

    sed -i 's:\\\${:\\\$\\{:' intltool-update.in
构建：
```
./configure --prefix=/usr
make
make check
make install
install -v -Dm644 doc/I18N-HOWTO /usr/share/doc/intltool-0.51.0/I18N-HOWTO
```
## Autoconf-2.71
```
tar -xvf autoconf-2.71.tar.xz
cd autoconf-2.71
make
make check TESTSUITEFLAGS=-j5
```
这里检查也要开多线程，因为测试时间会很长。
## Automake-1.16.4
```
tar -xvf automake-1.16.4.tar.xz
cd automake-1.16.4
./configure --prefix=/usr --docdir=/usr/share/doc/automake-1.16.4
make
make -j4 check
make install
```
此处检查也要开多线程，而且时间要长的多。
## Kmod-29
```
tar -xvf kmod-29.tar.xz
cd kmod-29
```
configure
```
./configure --prefix=/usr          \
            --sysconfdir=/etc      \
            --with-xz              \
            --with-zstd            \
            --with-zlib
```
编译

    make
该包的测试因为依赖`git`，因此跳过。    
安装该软件包，并创建与 Module-Init-Tools (曾经用于处理 Linux 内核模块的软件包) 兼容的符号链接： 
```
make install

for target in depmod insmod modinfo modprobe rmmod; do
  ln -sfv ../bin/kmod /usr/sbin/$target
done

ln -sfv kmod /usr/bin/lsmod
```
## Elfutils-0.185 中的 Libelf
libelf 在包`elfutils`中
```
tar -xvf elfutils-0.185.tar.bz2
cd elfutils-0.185
```
configure
```
./configure --prefix=/usr                \
            --disable-debuginfod         \
            --enable-libdebuginfod=dummy
```
编译，检查：
```
make
make check
```
我编译时此处会报错`FAIL: run-backtrace-native.sh`。该错误已于 11 月 14 日提交于 Gentoo 的 bug 处理，但版本是 0.186。不过这里只需要`libelf`，所以问题不大。    
> 其实跳过就好了……

只安装`libelf`：
```
make -C libelf install
install -vm644 config/libelf.pc /usr/lib/pkgconfig
rm /usr/lib/libelf.a
```
## Libffi-3.4.2
该包构建时会对特定处理器进行优化。
```
tar -xvf libffi-3.4.2.tar.gz
cd libffi-3.4.2
```
configure
```
./configure --prefix=/usr          \
            --disable-static       \
            --with-gcc-arch=native \
            --disable-exec-static-tramp
```
编译，检查，安装：
```
make
make check
make install
```
## OpenSSL-1.1.1l
```
tar -xvf openssl-1.1.1l.tar.gz
cd openssl-1.1.1l
```
configure
```
./config --prefix=/usr         \
         --openssldir=/etc/ssl \
         --libdir=lib          \
         shared                \
         zlib-dynamic
```
编译，测试：
```
make
make test
```
已知测试`30-test_afalg.t`可能会失败，忽略即可。     
安装：
```
sed -i '/INSTALL_LIBS/s/libcrypto.a libssl.a//' Makefile
make MANSUFFIX=ssl install
```
为安装目录加上版本号：

    mv -v /usr/share/doc/openssl /usr/share/doc/openssl-1.1.1l
安装额外的文档（可选）：

    cp -vfr doc/* /usr/share/doc/openssl-1.1.1l
## Python-3.9.6
还是一样的别解压错了。
```
tar -xvf Python-3.9.6.tar.xz
cd Python-3.9.6
```
configure
```
./configure --prefix=/usr       \
            --enable-shared     \
            --with-system-expat \
            --with-system-ffi   \
            --with-ensurepip=yes \
            --enable-optimizations
```
编译，安装：
```
make
make install
```
此处不建议测试，但如果一定要，运行`make test`。    
安装预先格式化的文档（可选）：
```
install -v -dm755 /usr/share/doc/python-3.9.6/html 

tar --strip-components=1  \
    --no-same-owner       \
    --no-same-permissions \
    -C /usr/share/doc/python-3.9.6/html \
    -xvf ../python-3.9.6-docs-html.tar.bz2
```
> 没错，就是另外一个 python-doc 包。

## Ninja-1.10.2
```
tar -xvf ninja-1.10.2.tar.gz
cd ninja-1.10.2
```
默认情况下，`ninjia`会尽量使用多进程，且进程数是 CPU 逻辑核心数 +2，以下修改可手动指定线程数：

    export NINJAJOBS=4
或者使用环境变量`NINJAJOBS`：
```
sed -i '/int Guess/a \
  int   j = 0;\
  char* jobs = getenv( "NINJAJOBS" );\
  if ( jobs != NULL ) j = atoi( jobs );\
  if ( j > 0 ) return j;\
' src/ninja.cc
```
编译，测试：
```
python3 configure.py --bootstrap
./ninja ninja_test
./ninja_test --gtest_filter=-SubprocessTest.SetWithLots
```
安装：
```
install -vm755 ninja /usr/bin/
install -vDm644 misc/bash-completion /usr/share/bash-completion/completions/ninja
install -vDm644 misc/zsh-completion  /usr/share/zsh/site-functions/_ninja
```
## Meson-0.59.1
```
tar -xvf meson-0.59.1.tar.gz
cd meson-0.59.1
python3 setup.py build
python3 setup.py install --root=dest
cp -rv dest/* /
install -vDm644 data/shell-completions/bash/meson /usr/share/bash-completion/completions/meson
install -vDm644 data/shell-completions/zsh/_meson /usr/share/zsh/site-functions/_meson
```
这个包没有测试。
## Coreutils-8.32
需要打补丁。
```
tar -xvf coreutils-8.32.tar.xz
cp coreutils-8.32-i18n-1.patch coreutils-8.32
cd coreutils-8.32
patch -Np1 -i coreutils-8.32-i18n-1.patch
```
configure
```
autoreconf -fiv
FORCE_UNSAFE_CONFIGURE=1 ./configure \
            --prefix=/usr            \
            --enable-no-install-program=kill,uptime
```
编译：

    make
测试（可选）：
```
make NON_ROOT_USERNAME=tester check-root
echo "dummy:x:102:tester" >> /etc/group
chown -Rv tester . 
su tester -c "PATH=$PATH make RUN_EXPENSIVE_TESTS=yes check"
sed -i '/dummy/d' /etc/group
```
安装：

    make install
将程序移动到 FHS 要求的位置： 
```
mv -v /usr/bin/chroot /usr/sbin
mv -v /usr/share/man/man1/chroot.1 /usr/share/man/man8/chroot.8
sed -i 's/"1"/"8"/' /usr/share/man/man8/chroot.8
```
## Check-0.15.2
```
tar -xvf check-0.15.2.tar.gz
cd check-0.15.2
./configure --prefix=/usr --disable-static
make
make check
make docdir=/usr/share/doc/check-0.15.2 install
```
## Diffutils-3.8
```
tar -xvf diffutils-3.8.tar.xz
cd diffutils-3.8
./configure --prefix=/usr
make
make install
```
## Gawk-5.1.0
```
tar -xvf gawk-5.1.0.tar.xz
cd gawk-5.1.0
```
确保构建时不安装某些文件：

    sed -i 's/extras//' Makefile.in
构建：
```
./configure --prefix=/usr
make
make check
make install
```
安装文档（可选）：
```
mkdir -v /usr/share/doc/gawk-5.1.0
cp    -v doc/{awkforai.txt,*.{eps,pdf,jpg}} /usr/share/doc/gawk-5.1.0
```
## Findutils-4.8.0
```
tar -xvf findutils-4.8.0.tar.xz
cd findutils-4.8.0
./configure --prefix=/usr --localstatedir=/var/lib/locate
make
chown -Rv tester .
su tester -c "PATH=$PATH make check"
make install
```
## Groff-1.22.4
```
tar -xvf groff-1.22.4.tar.gz
cd groff-1.22.4
```
configure，并将默认纸张改为 A4：

    PAGE=A4 ./configure --prefix=/usr
编译，安装,
此处编译强制`-j1`：
```
make -j1
make install
```
## GRUB-2.06
因为使用 UEFI，因此手册的安装流程不适用。    
对于 BIOS，这里给出命令（不包括解压）：     
**以下安装命令不支持 UEFI ！**
```
./configure --prefix=/usr          \
            --sysconfdir=/etc      \
            --disable-efiemu       \
            --disable-werror
make
make install
mv -v /etc/bash_completion.d/grub /usr/share/bash-completion/completions
make install
mv -v /etc/bash_completion.d/grub /usr/share/bash-completion/completions
```
## Gzip-1.10
```
tar -xvf gzip-1.10.tar.xz
cd gzip-1.10
make
make check
make install
```
## IPRoute2-5.13.0
```
tar -xvf iproute2-5.13.0.tar.xz
cd iproute2-5.13.0
```
不安装`arpd`：
```
sed -i /ARPD/d Makefile
rm -fv man/man8/arpd.8
```
禁用两个模块

    sed -i 's/.m_ipt.o//' tc/Makefile
编译，安装：
```
make
make SBINDIR=/usr/sbin install
```
安装文档（可选）：
```
mkdir -v              /usr/share/doc/iproute2-5.13.0
cp -v COPYING README* /usr/share/doc/iproute2-5.13.0
```
## Kbd-2.4.0
需要打补丁。
```
tar -xvf kbd-2.4.0.tar.xz
cp kbd-2.4.0-backspace-1.patch kbd-2.4.0
cd kbd-2.4.0
patch -Np1 -i kbd-2.4.0-backspace-1.patch
```
删除多余的`resizecons`程序及其 man 页面：
```
sed -i '/RESIZECONS_PROGS=/s/yes/no/' configure
sed -i 's/resizecons.8 //' docs/man/man8/Makefile.in
```
构建：
```
./configure --prefix=/usr --disable-vlock
make
make check
make install
```
安装文档（可选）：
```
mkdir -v            /usr/share/doc/kbd-2.4.0
cp -R -v docs/doc/* /usr/share/doc/kbd-2.4.0
```
## Libpipeline-1.5.3
```
tar -xvf libpipeline-1.5.3.tar.gz
cd libpipeline-1.5.3
./configure --prefix=/usr
make
make check
make install
```
## Make-4.3
```
tar -xvf make-4.3.tar.gz
cd make-4.3
./configure --prefix=/usr
make
make check
make install
```
检查是可选的。
## Patch-2.7.6
```
tar -xvf patch-2.7.6.tar.xz
cd patch-2.7.6
./configure --prefix=/usr
make
make check
make install
```
## Tar-1.34
```
tar -xvf tar-1.34.tar.xz
cd tar-1.34
```
configure
```
FORCE_UNSAFE_CONFIGURE=1  \
./configure --prefix=/usr
```
编译，检查：
```
make
make check
```
测试`capabilities: binary store/restore`的测试可能会失败或跳过。     
安装：
```
make install
make -C doc install-html docdir=/usr/share/doc/tar-1.34
```
## Texinfo-6.8
```
tar -xvf texinfo-6.8.tar.xz
cd texinfo-6.8
./configure --prefix=/usr
```
再次修复在使用 Glibc-2.34 或更新版本的系统上构建该软件包时出现的问题： 
```
sed -e 's/__attribute_nonnull__/__nonnull/' \
    -i gnulib/lib/malloc/dynarray-skeleton.c
```
编译，检查，安装：
```
make
make check
make install
```
安装 Tex 组件（可选）：

    make TEXMF=/usr/share/texmf install-tex
重更新 Info 文档（可选）：
```
pushd /usr/share/info
  rm -v dir
  for f in *
    do install-info $f dir 2>/dev/null
  done
popd
```
## Vim-8.2.3337
> 不会退出 vim 的去 BLFS 找其他编辑器

```
tar -xvf vim-8.2.3337.tar.gz
cd vim-8.2.3337
```
修改`vimrc`配置文件的默认位置为`/etc`： 

    echo '#define SYS_VIMRC_FILE "/etc/vimrc"' >> src/feature.h
编译：
```
./configure --prefix=/usr
make
```
测试：
```
chown -Rv tester .
su tester -c "LANG=en_US.UTF-8 make -j1 test" &> vim-test.log
```
此处输出被重定向到了文件，要自行 cat。测试成功完成后，日志文件末尾会包含 “ALL DONE”。     
安装：

    make install
此处手册将`vi`改为了指向`vim`的符号链接。虽然大部分发行版会单独安装`vi`，但毕竟现在没多少人会用`vi`吧。     
执行：
```
ln -sv vim /usr/bin/vi
for L in  /usr/share/man/{,*/}man1/vim.1; do
    ln -sv vim.1 $(dirname $L)/vi.1
done
```
创建文档的符号链接：

    ln -sv ../vim/vim82/doc /usr/share/doc/vim-8.2.3337
### Vim 配置
加入`/etc/vimrc`：
```
cat > /etc/vimrc << "EOF"
" Begin /etc/vimrc

" Ensure defaults are set before customizing settings, not after
source $VIMRUNTIME/defaults.vim
let skip_defaults_vim=1 

set nocompatible
set backspace=2
set mouse=
syntax on
if (&term == "xterm") || (&term == "putty")
  set background=dark
endif

" End /etc/vimrc
EOF
```
可根据自己喜好修改。虽然全局的这些就够了。     
## MarkupSafe-2.0.1
```
tar -xvf MarkupSafe-2.0.1.tar.gz
cd MarkupSafe-2.0.1
python3 setup.py build
python3 setup.py install --optimize=1
```
## Jinja2-3.0.1
```
tar -xvf Jinja2-3.0.1.tar.gz
cd Jinja2-3.0.1
python3 setup.py install --optimize=1
```
## Systemd-249
需要打补丁。
```
tar -xvf systemd-249.tar.gz
cp systemd-249-upstream_fixes-1.patch systemd-249
cd systemd-249
patch -Np1 -i systemd-249-upstream_fixes-1.patch
```
从默认的 udev 规则中删除不必要的组`render`和`sgx`： 
```
sed -i -e 's/GROUP="render"/GROUP="video"/' \
        -e 's/GROUP="sgx", //' rules.d/50-udev-default.rules.in
```
准备编译，包括创建`build`：
```
mkdir -p build
cd       build

LANG=en_US.UTF-8                    \
meson --prefix=/usr                 \
      --sysconfdir=/etc             \
      --localstatedir=/var          \
      --buildtype=release           \
      -Dblkid=true                  \
      -Ddefault-dnssec=no           \
      -Dfirstboot=false             \
      -Dinstall-tests=false         \
      -Dldconfig=false              \
      -Dsysusers=false              \
      -Db_lto=false                 \
      -Drpmmacrosdir=no             \
      -Dhomed=false                 \
      -Duserdb=false                \
      -Dman=false                   \
      -Dmode=release                \
      -Ddocdir=/usr/share/doc/systemd-249 \
      ..
```
编译，安装：
```
LANG=en_US.UTF-8 ninja
LANG=en_US.UTF-8 ninja install
```
安装 man 页面： 

    tar -xf ../../systemd-man-pages-249.tar.xz --strip-components=1 -C /usr/share/man
删除不必要的目录： 

    rm -rf /usr/lib/pam.d
初始化 machine id：

    systemd-machine-id-setup
设定启动目标单元的基本结构： 

    systemctl preset-all
禁用一个服务：

    systemctl disable systemd-time-wait-sync.service
## D-Bus-1.12.20
```
tar -xvf dbus-1.12.20.tar.gz
cd dbus-1.12.20
```
configure
```
./configure --prefix=/usr                        \
            --sysconfdir=/etc                    \
            --localstatedir=/var                 \
            --disable-static                     \
            --disable-doxygen-docs               \
            --disable-xml-docs                   \
            --docdir=/usr/share/doc/dbus-1.12.20 \
            --with-console-auth-dir=/run/console \
            --with-system-pid-file=/run/dbus/pid \
            --with-system-socket=/run/dbus/system_bus_socket
```
编译，安装：
```
make
make install
```
该包有测试，但需要一些不安装的依赖。      
创建一个符号链接：

    ln -sfv /etc/machine-id /var/lib/dbus
## Man-DB-2.9.4
```
tar -xvf man-db-2.9.4.tar.xz
cd man-db-2.9.4
```
cofigure
```
./configure --prefix=/usr                        \
            --docdir=/usr/share/doc/man-db-2.9.4 \
            --sysconfdir=/etc                    \
            --disable-setuid                     \
            --enable-cache-owner=bin             \
            --with-browser=/usr/bin/lynx         \
            --with-vgrind=/usr/bin/vgrind        \
            --with-grap=/usr/bin/grap
```
编译，检查，安装：
```
make
make check
make install
```
## Procps-ng-3.3.17
```
tar -xvf procps-ng-3.3.17.tar.xz
cd procps-3.3.17
```
这里解压出来的目录是`procps-3.3.17`，不是`procps-ng-3.3.17`。     
configure
```
./configure --prefix=/usr                            \
            --docdir=/usr/share/doc/procps-ng-3.3.17 \
            --disable-static                         \
            --disable-kill                           \
            --with-systemd
```
编译，测试，安装：
```
make
make check
make install
```
检查是可选的，且已知五项与 pkill 相关的测试可能失败。
## Util-linux-2.37.2
```
tar -xvf util-linux-2.37.2.tar.xz
cd util-linux-2.37.2
```
configure
```
./configure ADJTIME_PATH=/var/lib/hwclock/adjtime   \
            --libdir=/usr/lib    \
            --docdir=/usr/share/doc/util-linux-2.37.2 \
            --disable-chfn-chsh  \
            --disable-login      \
            --disable-nologin    \
            --disable-su         \
            --disable-setpriv    \
            --disable-runuser    \
            --disable-pylibmount \
            --disable-static     \
            --without-python     \
            runstatedir=/run
```
编译：

    make
测试（可选）：
```
rm tests/ts/lsns/ioctl_ns
chown -Rv tester .
su tester -c "make -k check"
```
安装：

    make install
## E2fsprogs-1.46.4
```
tar -xvf e2fsprogs-1.46.4.tar.gz
cd e2fsprogs-1.46.4
```
创建`build`：
```
mkdir -v build
cd       build
```
configure
```
../configure --prefix=/usr           \
             --sysconfdir=/etc       \
             --enable-elf-shlibs     \
             --disable-libblkid      \
             --disable-libuuid       \
             --disable-uuidd         \
             --disable-fsck
```
编译，测试，安装：
```
make
make check
make install
```
已知测试`u_direct_io`可能在一些系统上失败。      
删除无用的静态库： 

    rm -fv /usr/lib/{libcom_err,libe2p,libext2fs,libss}.a
更新系统`dir`文件：
```
gunzip -v /usr/share/info/libext2fs.info.gz
install-info --dir-file=/usr/share/info/dir /usr/share/info/libext2fs.info
```
安装额外文档（可选）：
```
makeinfo -o      doc/com_err.info ../lib/et/com_err.texinfo
install -v -m644 doc/com_err.info /usr/share/info
install-info --dir-file=/usr/share/info/dir /usr/share/info/com_err.info
```
## efivar-37 
这是 efibootmanager 的一个依赖。   
下载连接：https://github.com/rhboot/efivar/releases/download/37/efivar-37.tar.bz2   
需要补丁，连接：https://www.linuxfromscratch.org/patches/blfs/11.0/efivar-37-gcc_9-1.patch
```
tar -xvf efivar-37.tar.bz2
cp efivar-37-gcc_9-1.patch efivar-37
cd efivar-37
patch -Np1 -i efivar-37-gcc_9-1.patch
make install LIBDIR=/usr/lib
```
## Popt-1.18 
这是 efibootmanager 的一个依赖。     
下载连接：http://ftp.rpm.org/popt/releases/popt-1.x/popt-1.18.tar.gz
```
tar -xvf popt-1.18.tar.gz
cd popt-1.18
./configure --prefix=/usr --disable-static
make
make install
```

## efibootmgr-17 
这是 GRUB 用于支持 UEFI 的一个依赖。需要单独从 BLFS 上下载。      
下载连接：https://github.com/rhboot/efibootmgr/archive/17/efibootmgr-17.tar.gz
```
tar -xvf efibootmgr-17.tar.gz
cd efibootmgr-17
```
修复一个编译时会遇到的错误：

    sed -e '/extern int efi_set_verbose/d' -i src/efibootmgr.c
编译，安装：
```
make EFIDIR=/boot/efi/EFI EFI_LOADER=grubx64.efi
make install EFIDIR=/boot/efi/EFI
```
## FreeType-2.11.0 
这也是一个 GRUB 的依赖……      
而且虽然在 BLFS 里写着不是必须，但用 BLFS 的命令 configure 时会报错需要该字体。   
下载连接：https://downloads.sourceforge.net/freetype/freetype-2.11.0.tar.xz
```
tar -xvf freetype-2.11.0.tar.xz
cd freetype-2.11.0
```
构建：
```
sed -ri "s:.*(AUX_MODULES.*valid):\1:" modules.cfg &&

sed -r "s:.*(#.*SUBPIXEL_RENDERING) .*:\1:" \
    -i include/freetype/config/ftoption.h  &&

./configure --prefix=/usr --enable-freetype-config --disable-static &&
make
```
安装：

    make install
## GRUB-2.06 带 UEFI 支持
这里需要一个字体文件。    
下载连接：https://unifoundry.com/pub/unifont/unifont-13.0.06/font-builds/unifont-13.0.06.pcf.gz
```
tar -xvf grub-2.06.tar.xz
cd grub-2.06
```
安装字体：
```
mkdir -pv /usr/share/fonts/unifont &&
gunzip -c ../unifont-13.0.06.pcf.gz > /usr/share/fonts/unifont/unifont.pcf
```
取消编译用环境变量（虽然没设置）：

    unset {C,CPP,CXX,LD}FLAGS
configure
```
./configure --prefix=/usr        \
            --sysconfdir=/etc    \
            --disable-efiemu     \
            --enable-grub-mkfont \
            --with-platform=efi  \
            --disable-werror
```
编译：

    make
安装：
```
make install
mv -v /etc/bash_completion.d/grub /usr/share/bash-completion/completions
```

# 移除调试符号（可选）
> 毕竟大部分人都不会调试这些软件包吧。

```
save_usrlib="$(cd /usr/lib; ls ld-linux*)
             libc.so.6
             libthread_db.so.1
             libquadmath.so.0.0.0 
             libstdc++.so.6.0.29
             libitm.so.1.0.0 
             libatomic.so.1.2.0" 

cd /usr/lib

for LIB in $save_usrlib; do
    objcopy --only-keep-debug $LIB $LIB.dbg
    cp $LIB /tmp/$LIB
    strip --strip-unneeded /tmp/$LIB
    objcopy --add-gnu-debuglink=$LIB.dbg /tmp/$LIB
    install -vm755 /tmp/$LIB /usr/lib
    rm /tmp/$LIB
done

online_usrbin="bash find strip"
online_usrlib="libbfd-2.37.so
               libhistory.so.8.1
               libncursesw.so.6.2
               libm.so.6
               libreadline.so.8.1
               libz.so.1.2.11
               $(cd /usr/lib; find libnss*.so* -type f)"

for BIN in $online_usrbin; do
    cp /usr/bin/$BIN /tmp/$BIN
    strip --strip-unneeded /tmp/$BIN
    install -vm755 /tmp/$BIN /usr/bin
    rm /tmp/$BIN
done

for LIB in $online_usrlib; do
    cp /usr/lib/$LIB /tmp/$LIB
    strip --strip-unneeded /tmp/$LIB
    install -vm755 /tmp/$LIB /usr/lib
    rm /tmp/$LIB
done

for i in $(find /usr/lib -type f -name \*.so* ! -name \*dbg) \
         $(find /usr/lib -type f -name \*.a)                 \
         $(find /usr/{bin,sbin,libexec} -type f); do
    case "$online_usrbin $online_usrlib $save_usrlib" in
        *$(basename $i)* ) 
            ;;
        * ) strip --strip-unneeded $i 
            ;;
    esac
done

unset BIN LIB save_usrlib online_usrbin online_usrlib
```
该命令会报错一堆文件格式无法识别，可以安全忽略。
# 清理
现在所有包都安装完成了。是时候清理了。

    rm -rf /tmp/*
然后登出，重新 chroot：

    logout
```
chroot "$LFS" /usr/bin/env -i          \
    HOME=/root TERM="$TERM"            \
    PS1='(lfs chroot) \u:\w\$ '        \
    PATH=/usr/bin:/usr/sbin            \
    /bin/bash --login
```
这次 chroot 参数有点变化，之后 chroot 都需要执行现在改过的命令。
接着清理：
```
find /usr/lib /usr/libexec -name \*.la -delete
find /usr -depth -name $(uname -m)-lfs-linux-gnu\* | xargs rm -rf
```
最后，移除用户`tester`：

    userdel -r tester

至此，系统构建部分正式结束。