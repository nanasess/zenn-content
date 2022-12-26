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

### 2022-09-23追記

[0.67.6](https://github.com/microsoft/WSL/releases/tag/0.67.6) より、 [systemd が利用できるようになった](https://devblogs.microsoft.com/commandline/systemd-support-is-now-available-in-wsl/)。
以下の手順で systemd を利用可能

1. `stage3-amd64-systemd-YYYYMMDDTHHMMSSZ.tar.xz` をダウンロード
2. `wsl --import` を実行する
   ```
   wsl --import Gentoo C:\Users\nanasess\wsl\Gentoo\ .\stage3-amd64-systemd-20220206T170533Z.tar --version 2
   ```
3. `/etc/wsl.conf` に以下を追記する
   ```
   [boot]
   systemd=true
   ```
4. `wsl --shutdown` を実行し、 WSL2 を再起動する
5. `systemctl list-units --type=service` を実行して、サービス一覧が確認できれば OK

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

### equery

Portage の管理を簡単にするツール

https://wiki.gentoo.org/wiki/Equery/ja

```
emerge --ask app-portage/gentoolkit
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

- `x11-misc/xvfb-run` と `media-video/ffmpeg` に依存するライブラリが入っていれば、 Playwright の video を有効にできることを確認
  - `--headed` で実行する必要がある

```
## MAKEOPTS="-j28" とかだとビルド中にプロセスが落ちる
sudo MAKEOPTS="-j2" ACCEPT_KEYWORDS="~amd64" emerge --ask dev-lang/spidermonkey
sudo emerge --ask x11-misc/xvfb-run
sudo emerge --ask media-video/ffmpeg
```

## OpenRC から systemd へ移行する

[OpenRCなGentoo Linuxをsystemd化する](https://suzuki-kengo.net/openrc-systemd/) を参考にさせていただきました。

WSL2 の場合はWSLのカーネルを使用するため、カーネルの再構築は必要ない。
プロファイルを変更し、 `emerge @world` を実行すれば利用できる。

1. eselectコマンドを使用して、systemd用のプロファイルに切り替える
    ```
    eselect profile list　　　　#現在のプロファイル確認＆sytemdプロファイルの番号を確認。
    eselect profile set 15      #変更
    eselect profile list 　　   #確認
    -------(省略）---------
    [15]  default/linux/amd64/17.1/systemd (stable) *
    ```
2. 変更したプロファイルに基づき、システムを再構築する
    ```
    emerge -avDN @world
    ```
3. `/etc/wsl.conf` に以下を追記する
   ```
   [boot]
   systemd=true
   ```
4. `wsl --shutdown` を実行し、 WSL2 を再起動する
5. `systemctl list-units --type=service` を実行して、サービス一覧が確認できれば OK

## すべてのパッケージをリビルドする

```
sudo emerge -e world && emerge --depclean -a
```

see https://www.reddit.com/r/Gentoo/comments/hg9dit/comment/fw5su2t/?utm_source=share&utm_medium=web2x&context=3

### See Also
- https://suzuki-kengo.net/openrc-systemd/

