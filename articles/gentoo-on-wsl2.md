---
title: "Gentoo on WSL2"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Gentoo", "WSL2"]
published: true
---
*個人的なおぼえがきです*

https://wiki.gentoo.org/wiki/Gentoo_in_WSL

## セットアップ

例
https://ftp.jaist.ac.jp/pub/Linux/Gentoo//releases/amd64/autobuilds/current-stage3-amd64-openrc/stage3-amd64-openrc-20220206T170533Z.tar.xz

stage3-amd64-openrc-YYYYMMDDTHHMMSSZ.tar.xz をダウンロードする

Stage3 をインポートするだけでインストールできる
```
wsl --import Gentoo C:\Users\nanasess\wsl\Gentoo\ .\stage3-amd64-openrc-20220206T170533Z.tar --version 2
```

## ユーザー作成とパスワード設定

```shell
useradd -m -G wheel nanasess
passwd nanasess
passwd
```

## デフォルトユーザー

```config:/etc/wsl.conf
[user]
default=nanasess
```

https://developer.moe/gentoo-on-wsl-2

```config:/etc/portage/make.conf
USE="X gtk -ipv6 xml truetype pcre unicode embed cgi acl berkdb bzip2 calendar cdb cjk curl exif firebird ftp gmp iconv imap iodbc kerberos ldap mysqli nls posix postgres readline selinux session snmp soap sockets spell sqlite ssl tidy xmlrpc xpm zip zlib icu xft jpeg json imagemagick tiff svg xml harfbuzz gif libxml2 gd png webp"

# Enable python 3.9 and set 3.9 as default
PYTHON_TARGETS="python3_8 python3_9"
PYTHON_SINGLE_TARGET="python3_9"

PHP_TARGETS="php7-4"

# Define targets for QEMU
QEMU_SOFTMMU_TARGETS="aarch64 arm i386 riscv32 riscv64 x86_64"
QEMU_USER_TARGETS="aarch64 arm i386 riscv32 riscv64 x86_64"

# No hardware videocard support
VIDEO_CARDS="dummy"

# Disable non-functional sandboxing features
FEATURES="-ipc-sandbox -pid-sandbox -mount-sandbox -network-sandbox"

# Always ask when managing packages, always consider deep dependencies (slow)
EMERGE_DEFAULT_OPTS="--ask --complete-graph"

# Enable optimizations for the used CPU
COMMON_FLAGS="-march=znver2 -O2 -pipe"
CHOST="x86_64-pc-linux-gnu"
CFLAGS=${COMMON_FLAGS}
CXXFLAGS=${COMMON_FLAGS}
FCFLAGS=${COMMON_FLAGS}
FFLAGS=${COMMON_FLAGS}
MAKEOPTS="-j28"

# NOTE: This stage was built with the bindist Use flag enabled
PORTDIR=/var/db/repos/gentoo
DISTDIR=/var/cache/distfiles
PKGDIR=/var/cache/binpkgs

# This sets the language of build output to English.
# Please keep this setting intact when reporting bugs.
LC_MESSAGES=C
ACCEPT_LICENSE="*"
GENTOO_MIRRORS="https://ftp.riken.jp/Linux/gentoo/ rsync://ftp.riken.jp/gentoo/"
#GENTOO_MIRRORS="rsync://ftp.riken.jp/gentoo/"

```

## @world パッケージのインストール

```shell
emerge-webrsync
emerge --oneshot --deep sys-devel/gcc
emerge --oneshot --usepkg=n sys-devel/libtool
emerge --oneshot --emptytree --deep @world
emerge --oneshot --deep @preserved-rebuild
emerge --ask --depclean
```

## locale の設定

```shell
sudo echo "ja_JP.UTF-8 UTF-8" >> /etc/locale.gen
sudo locale-gen
sudo eselect locale list

```

## wslg の使用

```shell
export DISPLAY=:0.0
```

## emerge

```shell
emerge --ask app-admin/sudo
sudo emerge --ask app-editors/gvim
sudo emerge --ask app-shells/zsh
sudo emerge --ask app-shells/zsh-complations
sudo emerge --ask app-shells/gentoo-zsh-completions
sudo emerge --ask dev-lang/php:7.4
sudo emerge --ask net-libs/nodejs
sudo npm install --global yarn
sudo emerge --ask github-cli
sudo emerge --ask www-client/google-chrome
```

### Emacs

```shell
sudo emerge --ask app-editors/emacs
sudo emerge --ask media-fonts/mix-mplus-ipa
fc-cache -f -v
sudo emerge --ask sys-apps/ripgrep
sudo emerge --ask app-text/cmigemo
sudo emerge --ask app-arch/sharutils
sudo emerge --ask x11-apps/xrdb
echo "Emacs.font: MigMix 1M-12" >> ~/.Xresources
xrdb -merge ~/.Xresources
```

### OneDrive

```shell
emerge --ask --noreplace app-eselect/eselect-repository
sudo eselect repository enable dlang
sudo emerge --ask app-portage/layman
sudo layman -a dlang
sudo emerge --sync dlang
sudo emerge --ask x11-libs/libnotify
sudo emerge --ask net-misc/onedrive
onedrive --synchronize --single-directory 'emacs'
```

## Playwright で必要なライブラリ

https://github.com/microsoft/playwright/blob/main/packages/playwright-core/src/utils/nativeDeps.ts
