---
title: "glibc 2.39+ で半角になった△→○●を locale-eaw で直す (WSL2 + WezTerm + Emacs + Nix)"
emoji: "△"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["linux", "wezterm", "emacs", "nix", "unicode"]
published: true
---

:::message
この記事は [Claude Code](https://claude.com/claude-code) を使用して調査・実装・執筆しています。
:::

:::message alert
**2026-09 追記**: この記事の 3 層構成は「ターミナル側で Ambiguous 文字を全角にできる」ことが前提です。その後メインターミナルを Ghostty 系の [noctty](https://github.com/amanthanvi/noctty) に移したところ、この前提が崩れました。Ghostty 系を使っている方は、末尾の「追記 (2026-09)」の節を先に読んでください。
:::

## 何が起きたか

WSL2 Gentoo Linux (glibc 2.42) の環境で、ターミナルや Emacs の記号が半角表示になっていることに気づきました。

```
△→○●■□▲  ← 全部半角になっている
```

以前は全角 (2セル幅) で表示されていたはずの記号が、半角 (1セル幅) に変わっています。

### 環境構成

```
Windows 11
├── WezTerm (Windows 側で動作するターミナルエミュレータ)
│   └── WSL2 Gentoo Linux (glibc 2.42)
│       ├── zsh + Powerlevel10k
│       └── Emacs 30 (WSLg 経由で GUI 表示)
```

WezTerm は Windows 上で動作しており、WSL2 の Linux 環境に接続しています。この構成が後述の設定方法に影響します。

## 原因: glibc 2.39 の仕様変更

これらの文字は Unicode の East Asian Width プロパティで **Ambiguous (A)** に分類されています。Ambiguous とは「文脈によって全角にも半角にもなり得る」文字のことです。

**glibc 2.38 以前**では、`ja_JP.UTF-8` ロケールにおいて Ambiguous 文字を**全角 (width=2)** として扱うオーバーライドがありました。しかし **glibc 2.39 以降**、Unicode 標準に厳密準拠する方針に変わり、このオーバーライドが削除されました。

```python
# glibc 2.42 での wcwidth() の返り値
import ctypes
libc = ctypes.CDLL(None)
libc.wcwidth.argtypes = [ctypes.c_wchar]
libc.wcwidth.restype = ctypes.c_int

print(libc.wcwidth('△'))  # => 1  (以前は 2)
print(libc.wcwidth('→'))  # => 1  (以前は 2)
print(libc.wcwidth('─'))  # => 1  (これは元々 1)
```

この変更により、日本語環境で全角表示が一般的だった記号 (△→○●■□▲☆★ など) が軒並み半角になります。

## 解決策: locale-eaw

[locale-eaw](https://github.com/hamano/locale-eaw) は、glibc のロケールを修正して `wcwidth()` の返り値を適切に設定するプロジェクトです。

2つのバリアントがあります:

| バリアント | 方針 | 特徴 |
|-----------|------|------|
| **EAW-FULLWIDTH** | 全 Ambiguous 文字を全角 | シンプルだが TUI アプリ (tmux 等) が崩れる |
| **EAW-CONSOLE** (推奨) | 文字ごとに個別判定 | 罫線 (─│) は半角、記号 (△→○●) は全角 |

EAW-CONSOLE が推奨です。罫線文字を半角のまま維持するため、プロンプトや TUI アプリの表示が崩れません。

## 全体のアーキテクチャ

ターミナル環境で文字幅を正しく表示するには、**3つのレイヤー**すべてで幅を一致させる必要があります。

```
locale-eaw EAW-CONSOLE
├── glibc wcwidth()     -- WSL2 Linux 側: zsh 等のカーソル位置計算
├── WezTerm cell_widths -- Windows 側: ターミナルの描画セル数
└── Emacs char-width-table -- WSL2 Linux 側: Emacs 内部の文字幅

UDEV Gothic JPDOC (全角グリフのフォント)
├── WezTerm -- Windows 側: プライマリフォント
└── Emacs   -- WSL2 Linux 側: プライマリフォント
```

WezTerm は Windows 側、zsh/Emacs は Linux 側で動作するため、**それぞれ別の方法で文字幅を設定する必要があります**。1つでもずれると、カーソル位置のずれや文字の重なりが発生します。

## 実装

### 1. locale-eaw のインストール (home-manager)

locale-eaw は通常 `/etc/locale.gen` を編集してシステムレベルで導入しますが、home-manager でユーザー空間に導入できます。これは **WSL2 Linux 側の glibc `wcwidth()` を修正**します。

```nix
# modules/locale-eaw/default.nix
{ config, pkgs, lib, ... }:

let
  eawCharmap = ./UTF-8-EAW-CONSOLE.gz;
  localeDir = "${config.home.homeDirectory}/.local/share/locale";
in
{
  # システムの glibc に含まれる localedef を使用する必要がある
  # Nix の localedef ではシステム glibc と互換性のないロケールデータが
  # 生成される可能性がある
  home.activation.localeEaw = lib.hm.dag.entryAfter [ "writeBoundary" ] ''
    mkdir -p "${localeDir}"
    charmap=$(${pkgs.coreutils}/bin/mktemp)
    ${pkgs.gzip}/bin/gunzip -c ${eawCharmap} > "$charmap"
    if [ -x /usr/bin/localedef ]; then
      /usr/bin/localedef -f "$charmap" -i ja_JP "${localeDir}/ja_JP.utf8" || \
        echo "warning: localedef failed" >&2
    else
      echo "warning: /usr/bin/localedef not found, skipping locale-eaw setup" >&2
    fi
    ${pkgs.coreutils}/bin/rm -f "$charmap"
  '';

  home.sessionVariables = {
    LOCPATH = "${localeDir}:/usr/lib/locale";
  };
}
```

ポイント:
- **`localedef` はシステムの `/usr/bin/localedef` を使う**: Nix の glibc でコンパイルするとシステム glibc と互換性のないロケールデータが生成される
- **ロケール名は `ja_JP.utf8` のまま**: charmap だけ `UTF-8-EAW-CONSOLE` に差し替え
- **`LOCPATH`** でユーザー空間のロケールをシステムより優先

これで WSL2 Linux 側の `wcwidth('△')` が 2 を返すようになります。

```bash
$ python3 -c "
import ctypes, locale
locale.setlocale(locale.LC_CTYPE, 'ja_JP.utf8')
libc = ctypes.CDLL(None)
libc.wcwidth.argtypes = [ctypes.c_wchar]
libc.wcwidth.restype = ctypes.c_int
print('△', libc.wcwidth('△'))  # => 2
print('→', libc.wcwidth('→'))  # => 2
print('─', libc.wcwidth('─'))  # => 1 (罫線は半角のまま)
"
```

### 2. WezTerm の設定 (Windows 側)

WezTerm は **Windows 側で動作**しているため、Linux の `LOCPATH` やフォント (`fc-list`) は参照できません。以下の 3点を Windows 側に配置する必要があります:

1. **JPDOC フォント** -- `font_dirs` で Windows 側のディレクトリを指定
2. **cell_widths 設定** -- `dofile` で Windows 側の Lua ファイルを読み込み
3. **WezTerm 設定** -- `/mnt/c/Users/<user>/.wezterm.lua` にコピー

locale-eaw は WezTerm 用の設定ファイル `eaw-console-wezterm.lua` を生成しています。

```lua
-- wezterm.lua (Windows 側 C:\Users\<username>\.wezterm.lua にコピーされる)
local wezterm = require("wezterm")
local config = wezterm.config_builder()

-- Windows 側のフォントディレクトリを指定 (JPDOC はここにコピー)
config.font_dirs = { 'C:\\Users\\<username>\\.local\\share\\fonts' }
-- JPDOC をプライマリフォント、NF を Nerd Font 用フォールバック
config.font = wezterm.font_with_fallback {
  'UDEV Gothic JPDOC',
  'UDEV Gothic NF',
}

-- locale-eaw EAW-CONSOLE に合わせた文字幅設定
local eaw = dofile(wezterm.config_dir .. '/.eaw-console-wezterm.lua')
config.cell_widths = eaw

return config
```

home-manager の activation でフォントと設定ファイルを Windows 側にコピーします:

```nix
# hosts/wsl-gentoo.nix
# WezTerm 設定を Windows 側にコピー
home.activation.weztermConfig = lib.hm.dag.entryAfter [ "writeBoundary" ] ''
  install -Dm644 ${../modules/wezterm/wezterm.lua} \
    /mnt/c/Users/${config.home.username}/.wezterm.lua
  install -Dm644 ${../modules/locale-eaw/eaw-console-wezterm.lua} \
    /mnt/c/Users/${config.home.username}/.eaw-console-wezterm.lua
'';

# JPDOC フォントを Windows 側にコピー
home.activation.weztermFonts = lib.hm.dag.entryAfter [ "writeBoundary" ] ''
  fontdir="/mnt/c/Users/${config.home.username}/.local/share/fonts"
  mkdir -p "$fontdir"
  install -m644 ${pkgs.udev-gothic}/share/fonts/udev-gothic/UDEVGothicJPDOC-*.ttf \
    "$fontdir/"
'';
```

#### なぜ `treat_east_asian_ambiguous_width_as_wide` ではダメか

WezTerm には `treat_east_asian_ambiguous_width_as_wide = true` というオプションがありますが、これは**全ての** Ambiguous 文字を全角にしてしまいます。罫線 (`─│`) も全角になり、プロンプトが崩れます。

`cell_widths` なら文字ごとに幅を指定できるため、EAW-CONSOLE と同じ「罫線は半角、記号は全角」を実現できます。

#### なぜ UDEV Gothic JPDOC か

UDEV Gothic NF は △→○● などの記号に**半角グリフ**を持っています (JetBrains Mono 由来)。`cell_widths` で 2セル確保しても、フォントのグリフが1セル分しかないため余白が出ます。

UDEV Gothic JPDOC は同じ記号に**全角グリフ**を持っています (BIZ UD Gothic 由来)。2セル確保 + 全角グリフで正しく描画されます。

```
フォント         △ のグリフ幅   cell_widths=2 での表示
UDEV Gothic NF   1024 (1セル)   [△ ]  ← 余白が出る
UDEV Gothic JPDOC 2048 (2セル)  [△]   ← 正しい
```

JPDOC は Nerd Font を含まないため、NF をフォールバックに設定して Nerd Font シンボルを補います。

JPDOC フォントは nixpkgs の `udev-gothic` パッケージに含まれています。NF zip (`UDEVGothic_NF_*.zip`) には含まれていないので注意してください。

### 3. Emacs の設定 (WSL2 Linux 側)

Emacs は WSLg 経由で GUI 表示されるため、**Linux 側のフォントと設定**を使います。WezTerm とは独立して設定が必要です。

#### char-width-table (eaw-console.el)

locale-eaw は Emacs 用の `eaw-console.el` も生成しています。`set-language-environment "Japanese"` が `char-width-table` をリセットするため、**その後に**読み込む必要があります。

```elisp
(set-language-environment "Japanese")
(set-default-coding-systems 'utf-8-unix)
;; ... 省略 ...

;; set-language-environment の後に読み込む
(load (expand-file-name
       (locate-user-emacs-file "site-lisp/eaw-console")) t t)
```

#### プライマリフォント

Emacs の `set-fontset-font` はフォールバック機構です。プライマリフォントにグリフがある場合は無視されます。UDEV Gothic NF に △→ のグリフがある (半角) ため、`set-fontset-font` で JPDOC を指定しても効きません。

**プライマリフォント自体を JPDOC にする**必要があります。

```elisp
(when (eq system-type 'gnu/linux)
  (defun my/set-font-linux (frame)
    (when (display-graphic-p frame)
      (set-face-attribute 'default frame
                          :family "UDEV Gothic JPDOC" :height 120)))
  (if (daemonp)
      (add-hook 'after-make-frame-functions #'my/set-font-linux)
    (when (display-graphic-p)
      (set-face-attribute 'default nil
                          :family "UDEV Gothic JPDOC" :height 120))))
```

### 4. home-manager での全体構成

```
flake.nix
  modules = [ ... ./modules/locale-eaw ./modules/emacs ... ];

modules/locale-eaw/
  default.nix              -- localedef + LOCPATH 設定 (Linux 側)
  UTF-8-EAW-CONSOLE.gz     -- locale-eaw の charmap
  eaw-console-wezterm.lua  -- WezTerm cell_widths (Windows 側にコピー)
  eaw-console.el           -- Emacs char-width-table (Linux 側)

hosts/wsl-gentoo.nix
  -- JPDOC フォントと設定ファイルを Windows 側にコピー
  home.activation.weztermConfig = ...;
  home.activation.weztermFonts = ...;
```

## 注意: Claude Code との互換性

Claude Code の TUI は Node.js の文字幅ライブラリ (標準 Unicode 幅テーブル) を使用しており、locale-eaw の幅設定とは一致しません。特に `●` (U+25CF) や `⎿` (U+23BF) で描画崩れが発生します。

WezTerm の `cell_widths` でこれらの文字だけ半角に戻すことで改善できます。

```lua
local eaw = dofile(wezterm.config_dir .. '/.eaw-console-wezterm.lua')
-- Claude Code TUI との互換性のため特定の記号を半角に戻す
table.insert(eaw, {first = 0x23bf, last = 0x23bf, width = 1})  -- ⎿
table.insert(eaw, {first = 0x25cf, last = 0x25cf, width = 1})  -- ●
config.cell_widths = eaw
```

この問題は [anthropics/claude-code#35560](https://github.com/anthropics/claude-code/issues/35560) で報告されています。

## 追記 (2026-09): Ghostty 系ターミナル (noctty) では、この 3 層構成は不要

この記事を書いたあと、メインターミナルを WezTerm から [noctty](https://github.com/amanthanvi/noctty) (Ghostty の Windows port fork) に移しました。結論から言うと、**Ghostty 系ターミナルを使う場合、glibc 層と端末層の設定はどちらも不要**です。入れたままにすると、むしろ不整合の原因になります。

### Ghostty 系は Ambiguous を幅 1 に固定している

Ghostty は文字幅の単一の情報源として `uucode` を使っており、East Asian Ambiguous を幅 1 として扱います。`src/build/uucode_config.zig` の `computeWidth`:

```zig
data.width = @min(2, data.wcwidth_standalone);
```

`wcwidth_standalone` を計算する uucode 側 (`src/x/config_x/wcwidth.zig`) は Wide (W) / Fullwidth (F) だけを 2 とし、Ambiguous (A) は 1 にします。ドキュメントコメントにも UAX #11 §5 の推奨が引用されています。

> Ambiguous width (A) characters are treated as width 1. Per UAX #11 §5 Recommendations: "If the context cannot be established reliably, they should be treated as narrow characters by default"

そして、これを**ユーザー設定で変える手段がありません**。

- `grapheme-width-method` (`unicode` / `legacy`) はグラフェムクラスタの幅計算方式の切替で、Ambiguous の判定には効きません。ドキュメントコメントにも「libc `wcswidth` を使うとは限らない」と明記されています。
- WezTerm の `cell_widths` や `treat_east_asian_ambiguous_width_as_wide` に相当する設定は存在しません (`adjust-cell-width` はセルのピクセル幅を調整するもので別物です)。
- そもそも Windows 側で動くプロセスなので、WSL 側の `LOCPATH` は参照されません。

### glibc だけ幅 2 のままにすると何が起きるか

ターミナルが幅 1 で描画しているのに、その上で動く Linux 側のプログラムが `wcwidth()` から 2 を受け取ると、**両者のカーソル位置の認識が 1 セルずつずれます**。

- zsh (Powerlevel10k の右プロンプトや複数行プロンプトの折り返し)
- readline (行編集中の再描画)
- tmux (ペイン境界やステータスラインの文字数計算)

実測 (glibc 2.43):

```
LOCPATH あり (この記事の構成): △→○●■□▲ = 2,  ─│ = 1,  あ = 2  ← 端末とずれる
LOCPATH なし:                  △→○●■□▲ = 1,  ─│ = 1,  あ = 2  ← 端末と一致
```

つまり Ghostty 系では、**端末に合わせて全部を幅 1 に統一する**のが唯一整合する選択肢になります。

### 変更後の構成: Emacs GUI だけ幅 2 を残す

一方、Emacs の GUI (WSLg) はターミナルのセルとは無関係な独立したレンダラなので、Emacs 内部だけ幅 2 にしても矛盾しません。日本語の文書で △○● が細くならない利点を取れるので、ここだけ残しました。

| レイヤー | 幅 | 設定 |
|---------|----|------|
| ターミナル (noctty / Ghostty 系) | 1 | uucode のテーブル (変更不可) |
| glibc `wcwidth()` (zsh / readline / tmux) | 1 | 素の `ja_JP.utf8` を使う (`LOCPATH` の上書きをやめる) |
| Claude Code TUI | 1 | Node.js の標準 Unicode 幅テーブル |
| Emacs GUI (WSLg) | 2 | `eaw-console.el` + UDEV Gothic JPDOC |
| Emacs TUI (`emacs -nw`) | 1 | `eaw-console.el` を読み込まない |

Emacs 側で 1 点だけ注意が要ります。`char-width-table` は**プロセスグローバル**でフレームごとには切り替えられないため、GUI 用の設定のまま `emacs -nw` を noctty 上で開くと、今度は Emacs だけが幅 2 前提で描画してずれます。そこで GUI (と daemon) のときだけ読み込むようにしました。

```elisp
;; daemon 起動時は初期化の時点で display-graphic-p が nil になるので daemonp も見る
(when (or (daemonp) (display-graphic-p))
  (load (expand-file-name
         (locate-user-emacs-file "site-lisp/eaw-console")) t t))
```

副次的に、`cell_widths` を捨てたことで「注意: Claude Code との互換性」で書いた `●` / `⎿` の例外処理も不要になりました。Claude Code の TUI はもともと Ambiguous を幅 1 として扱うので、端末側が幅 1 なら最初から一致します。

### 見た目のトレードオフ

Ghostty 系では Ambiguous がセル 1 個分になるため、記号の描画は文字ブロックによって挙動が分かれます。

| 文字 | ブロック | セル幅 | 描画 |
|------|---------|--------|------|
| `→` | Arrows (U+2190..U+21FF) | 1 | `.fit` 制約で 1 セル分に縮小される |
| `△○●■□▲` | Geometric Shapes (U+25A0..U+25FF) | 1 | 制約なし。全角グリフが隣のセルに視覚的にはみ出す |

Ghostty は記号ブロック (`isSymbol` が真のもの) にだけ「セル内に収める」制約をかけますが、Geometric Shapes はその対象に入っていません。ただし**セル管理側は 1 セルとして正しく進む**ので、カーソル位置・表・TUI のレイアウトは崩れません。見た目のはみ出しと引き換えに、レイアウトの整合性を取っている形です。

### 教訓: 文字幅は「一番融通の利かないレイヤー」に合わせる

この記事の 3 層構成が成立していたのは、WezTerm が `cell_widths` で幅を自由に決められたからです。端末側が Unicode 標準に固定していて設定を持たない場合、Linux 側だけを幅 2 に寄せても、ずれが「端末とアプリケーションの間」に移動するだけで解決しません。

- 端末が幅を設定できる (WezTerm 等) → 3 層すべてを幅 2 に揃えられる。この記事の構成が使えます
- 端末が幅 1 に固定 (Ghostty 系) → 全レイヤーを幅 1 に揃える。端末と独立したレンダラ (Emacs GUI 等) だけは個別に決めてよい

## まとめ

:::message
以下は WezTerm を前提とした構成です。Ghostty 系ターミナル (noctty など) を使う場合は、上の「追記 (2026-09)」を参照してください。
:::

glibc 2.39+ で East Asian Ambiguous 文字が半角になる問題は、以下の3層で対処できます。

| レイヤー | 動作環境 | 設定 | 効果 |
|---------|---------|------|------|
| glibc (`wcwidth`) | WSL2 Linux | locale-eaw + `LOCPATH` | zsh 等のカーソル位置が正確に |
| WezTerm | Windows | `cell_widths` + JPDOC フォント | ターミナル描画が正確に |
| Emacs | WSL2 Linux (WSLg) | `eaw-console.el` + JPDOC フォント | Emacs 内部の文字幅が正確に |

ポイントは「全レイヤーで幅を一致させる」ことです。特に WSL2 環境では **WezTerm (Windows) と zsh/Emacs (Linux) が別々の OS で動作**しているため、それぞれ独立した設定が必要です。

locale-eaw の EAW-CONSOLE バリアントを使えば、罫線は半角のまま維持されるため、TUI アプリやプロンプトの表示を崩さずに記号だけ全角にできます。

## 参考リンク

- [locale-eaw](https://github.com/hamano/locale-eaw) -- glibc ロケール修正
- [font-eaw](https://github.com/hamano/font-eaw) -- locale-eaw に最適化されたフォント
- [UDEV Gothic](https://github.com/yuru7/udev-gothic) -- JPDOC バリアントを含むプログラミングフォント
- [WezTerm cell_widths](https://wezterm.org/config/lua/config/cell_widths.html)
- [Unicode East Asian Width (TR#11)](https://www.unicode.org/reports/tr11/)
- [nanasess/home-manager#64](https://github.com/nanasess/home-manager/pull/64) -- locale-eaw 導入 PR
- [nanasess/home-manager#65](https://github.com/nanasess/home-manager/pull/65) -- Claude Code TUI 互換性改善 PR
- [noctty](https://github.com/amanthanvi/noctty) -- Ghostty の Windows port fork (追記で移行したターミナル)
- [nanasess/home-manager#71](https://github.com/nanasess/home-manager/issues/71) -- noctty 移行に伴う wcwidth 整合性の見直し Issue
- [nanasess/home-manager#150](https://github.com/nanasess/home-manager/pull/150) -- glibc / 端末レイヤーを撤去した PR
