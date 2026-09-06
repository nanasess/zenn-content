---
title: "WSL 用ターミナルは noctty が良いぞ — PR を 4 本マージしてもらって分かったこと"
emoji: "🌙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["noctty", "wsl", "terminal", "zig", "claudecode"]
published: true
---

:::message
この記事は [Claude Code](https://claude.com/claude-code) を使用して調査・実装・執筆しています。
:::

## 結論

Windows から WSL2 を常用しているなら、ターミナルは [noctty](https://github.com/amanthanvi/noctty) を勧めます。

- **Windows ネイティブとしての作り込みが厚い**。既定ターミナル登録、エクスプローラのコンテキストメニュー、ジャンプリスト、セッション復元、名前付きレイアウト、Quick select、省電力レンダリング
- **速い**。ただし**同梱 ConPTY を配置しないと本来の性能が出ない**。ここが最大の落とし穴です
- **メンテナが速い**。私が出した PR 4 本は全部マージされ、直近 2 本は**1 時間強**でした
- **AI 支援のコントリビュートが明文で歓迎されている**。`AI_POLICY.md` があり、条件も明確です

以下は、半年ほど Windows 側ターミナルを転々とした末に noctty へ移り、その過程で 4 本 PR を出した記録です。

## noctty とは

[Ghostty](https://ghostty.org/) の**ターミナルコアを Windows ネイティブアプリに載せたフォーク**です。Zig 製、MIT ライセンス、テレメトリなし。元は winghostty という名前でしたが、2026 年 8 月に Ghostty チームからの商標に関する要請を受けて `noctty` へ改名しました（[noctty#119](https://github.com/amanthanvi/noctty/issues/119)）。同一プロジェクト・同一メンテナで、旧 URL はリダイレクトされます。

Ghostty 本体の GPU レンダラ・フォント処理・設定体系・VT 実装をそのまま継承し、**その周りの Win32 アプリはこのフォークのために書き下ろされています**。バージョン番号も `major.minor` が upstream Ghostty の系列（現在 1.3 = upstream 1.3.2 ベース）、`patch` が noctty のリリース番号という付け方です。

### Windows で WSL 用ターミナルを選ぶときの地図

私が実際に触ったものだけ挙げます。

| | 種別 | WSL への接続 |
|---|---|---|
| Windows Terminal | Microsoft 純正 | ConPTY |
| WezTerm | クロスプラットフォーム | ConPTY |
| Ghostty Windows port | 本家の Windows 対応 PR | ConPTY |
| **noctty** | Windows ネイティブフォーク | ConPTY |
| GhostInTheWSL | WSL 専用フォーク | **Hyper-V ソケット + vsock で Linux PTY へ直結** |

最後の [GhostInTheWSL](https://github.com/Codavo/ghostinthewsl) だけが異質で、ConPTY を経由せず WSL2 の Linux PTY に直結します。私は長らくこれを使っていました。**ConPTY を挟まないほうが速く、エスケープシーケンスが素通しになる**からです。

そして「noctty は ConPTY 経由だから」という理由で、私は一度 noctty への移行を見送っていました。**この判断は間違っていました。**その話は後半でします。

## 機能面: Windows ネイティブならではの作り込み

まず何が入っているか。upstream Ghostty には無い、あるいは Windows 版だけの機能がかなりあります。

### Windows シェルとの統合

| 機能 | 内容 |
|---|---|
| **既定のターミナル登録** | `noctty +register-default-terminal` でユーザー単位の既定ターミナルになる。Windows Terminal 1.24 以降の OpenConsole handoff（`ITerminalHandoff3`）経由 |
| **エクスプローラのコンテキストメニュー** | 「Open noctty here」を右クリックメニューに追加 |
| **タスクバーのジャンプリスト** | 最近使ったディレクトリとプロファイルがタスクバーの右クリックに並ぶ |
| **統合タイトルバー** | Windows 11 ではアプリ所有のキャプション。ネイティブのキャプション操作と Snap Layout ホバーがタブバーと同居する |
| **昇格ウィンドウ** | 管理者権限のウィンドウを integrity-scoped な IPC で分離。昇格セッションは復元しない |
| **SSH ホスト検出** | `%USERPROFILE%\.ssh\config` の具体的なエイリアスを、シェルピッカーとコマンドパレットに `ssh.exe` の起動エントリとして並べる |

WSL 使いとして地味に効くのは既定ターミナル登録です。`wsl.exe` を他所から叩いたときも noctty で開くようになります。

### ターミナルとしての機能

| 機能 | 内容 |
|---|---|
| **セッション復元** | ウィンドウ・タブ・split・プロファイル・作業ディレクトリ・タイトルを `session-state.json` に永続化。`window-save-state-scrollback` でスクロールバックの平文スナップショットも（オプトイン）。子プロセスは復元されない |
| **名前付きレイアウト** | 1 ウィンドウのタブ / split / プロファイル / cwd / タイトルの形を保存し、キーバインド・パレット・CLI から新しいウィンドウとして再現する |
| **Quick select + copy mode** | 画面上の正規表現ターゲットをヒント表示して選択。モーダルなキーボード選択とスクロールバック移動 |
| **ユニバーサルパレット** | `Ctrl+Shift+P`。コマンド・プロファイル・レイアウトなどを 1 つのファジー検索面に混ぜて出す |
| **タブのドラッグ** | 同一ウィンドウ内の並べ替えに加えて、**特定のペインへドロップして split にする**操作ができる |
| **ネイティブ設定ウィンドウ** | GUI から設定を編集。書き戻しは元ファイルの構造を保つ |
| **スクロールバー** | ペインごとのグラフィカルスクロールバー。検索ヒットがマーカーとして出る |
| **カスタムシェーダー** | v1.3.120 で同梱 |

### 省電力と性能

- **Power-aware rendering** — `unfocused-render-fps` で背面ウィンドウの提示レートを抑え、`power-saver-rendering` でバッテリー時のペースを制御。**最小化されたウィンドウと DWM にクロークされたウィンドウは present しない**
- **CI にベンチマークスイートとリグレッションゲート**がある（[#121](https://github.com/amanthanvi/noctty/issues/121) / PR #191）。性能劣化が CI で止まる

ノート PC で使うとき、背面のターミナルが GPU を焼き続けないのは効きます。

### 運用・プライバシー

- **テレメトリなし**。外向き通信は GitHub Releases への更新チェックのみ（起動時、最大 24 時間に 1 回）。`auto-update = off` で止まる。更新は必ずユーザーが開始する
- **クラッシュダンプはローカルのみ**。`%LOCALAPPDATA%\noctty\crash` に置かれ、アップロードされない。`noctty +crash-report` で読める
- **`--safe-mode`** — 壊れた設定や保存セッションで起動できなくなったとき、組み込みデフォルトで 1 回だけ起動する
- リリースには**署名済みマニフェストと SHA256 チェックサム**が付く。ポータブル ZIP の自動更新は、全ペイロードファイルを網羅した pin 済み発行者署名マニフェストが無いと適用に進まない
- x64 と **ARM64** の両方

自己署名証明書なので SmartScreen は初回に警告します。そこは checksum で確認する運用です。

### ドキュメントが正直

個人的にいちばん好感を持ったのがこれです。[`docs/windows-capability-matrix.md`](https://github.com/amanthanvi/noctty/blob/main/docs/windows-capability-matrix.md) が、upstream Ghostty のドキュメント項目を 1 行ずつ拾って `supported` / `partial` / `windows-specific` に分類しています。

そして `partial` の書き方が具体的です。

> **アクセシビリティ**: UI Automation は日常的なクロームと端末テキストをカバーするが、キャプションボタン・オーバーレイ行・メニューは未カバー。**測定したのは NVDA のみで、結果はまちまち**。
>
> **`link-previews`**: リンクのマッチ・ハイライト・オープンは動くが、**Win32 ランタイムはプレビューのツールチップを描画しない**。
>
> **`background-blur`**: Windows 11 22H2 以降で `background-opacity < 1` のとき DWM の tabbed backdrop を要求する。**Windows 10 と 11 21H2 では受理されるが何も起きない**。半径は on/off として扱う。

「対応しています」で濁さず、どこまで動いてどこから動かないかが書いてある。Windows Terminal と Git Bash/mintty からの移行ガイドも別途あります。

## 発見: 「ConPTY だから遅い」は半分間違っていた

ここからが本題です。

### ConPTY には世代がある

ConPTY は Windows のコンソールホスト（`conhost`）が提供する疑似端末ですが、**2 つの世代があります**。境界は [microsoft/terminal#17510](https://github.com/microsoft/terminal/pull/17510) で、noctty のドキュメント（[`docs/windows-vt-conformance.md`](https://github.com/amanthanvi/noctty/blob/main/docs/windows-vt-conformance.md)）が両者を整理しています。

| | 挙動 |
|---|---|
| **ConPTY v1**<br>(Windows Terminal 1.21 以前、および検証されたすべての in-box conhost) | 子プロセスの VT を conhost のテキストバッファへ**パースし、スナップショットを再レンダリング**してターミナルのパイプへ流す。未知の、あるいは不完全にモデル化されたシーケンスは**落ちる・並べ替わる・別物として合成される** |
| **ConPTY v2**<br>(Windows Terminal 1.22+ と再頒布物) | コンソール状態のためにパースしつつ、**元の VT バイトを別途そのままパイプへ書く**。直接転送 |

**「ConPTY だから遅い / シーケンスが落ちる」という定説は v1 の話**でした。v2 は素通しします。

そして重要な注意が書かれています。**in-box conhost の世代を OS のビルド番号から推測してはいけない**。v2 を含む最初の Windows ビルドは特定できていない、と。

### 実際に測ってみた

自分の環境で何が起きているか、確認スクリプトを書きました。端末に問い合わせシーケンスを投げて応答を読むだけのものです。

```bash
# kitty graphics に対応しているか問い合わせる (1x1 の画像を投げて OK が返るか)
printf '\033_Gi=31,s=1,v=1,a=q,t=d,f=24;AAAA\033\\'
```

結果です。

```
--- T1: Primary DA ---
  応答: ^[[?61;6;7;21;22;23;24;28;32;42c

--- T2: XTVERSION ---
  応答: ^[P>|ghostty 1.3.2-dev+windows^[\

--- T3: kitty graphics クエリ (a=q) ---
  応答: (なし)          ← 画像プロトコルが通らない
```

**T1 の応答が決定的でした。** noctty（Ghostty コア）自身の Primary DA 応答は `src/termio/stream_handler.zig` にハードコードされていて `\x1B[?62;22;52c` です。ところが返ってきたのは `?61;6;7;...` という別物、つまり **conhost が問い合わせを横取りして自分で答えていた**わけです。

シーケンスごとに扱いが 3 通りに分かれていました。

| シーケンス | in-box conhost (v1) の扱い |
|---|---|
| Primary DA (`ESC [ c`) | **横取りして自分で答える**（端末まで届かない） |
| XTVERSION (`ESC [ > q`) | 素通し |
| **APC（kitty graphics）** | **握り潰し** |

同じスクリプトを GhostInTheWSL（vsock 直結）で走らせると T3 は `ESC _ Gi=31;OK ESC \` を返し、画像も表示されました。**差はトランスポートだけ**なので、ConPTY が APC を落としていることが確定します。

なお noctty のドキュメントには、これを**子プロセス側から測った**カタログも載っています。子に `ENABLE_VIRTUAL_TERMINAL_PROCESSING` を明示的に立て、マーカーとペイロードを `WriteFile` で書いて出口のパイプを読む、という測り方です。

| シーケンス | 同梱 v2 | in-box v1 |
|---|---|---|
| Kitty graphics APC | **バイト単位で一致** | **空** |
| Sixel DCS | **バイト単位で一致** | **空** |

印字可能なマーカーは**両方で生き残っている**ので、空スライスは「パイプが空だった」でも「子プロセスが失敗した」でもなく、**シーケンスが剥がされた証拠**になります。

v1 のシーケンス別の挙動も整理されていて、これがなかなか強烈です。

- **APC / PM / SOS は無視される**（[microsoft/terminal#7340](https://github.com/microsoft/terminal/pull/7340)）
- **DCS / Sixel** は 1.13 まで破棄、1.14〜1.21 はホワイトリストのみ転送
- **OSC 10/11/12 の色問い合わせは飲み込まれる**。端末は見ることも答えることもできない
- **OSC 8 ハイパーリンクは再合成される**。`id=` は書き換えられ、ID の無いものには PID ベースの合成 ID が付き、`id` 以外のパラメータキーは落ち、BEL 終端は ST になる
- 同期出力（`CSI ?2026h/l`）はレンダラのスナップショットが非同期にフラッシュされる関係で**順序が入れ替わりうる**

### noctty には公式の回避策があった

ソースを読んでいて、こういう文字列を見つけました（`src/pty.zig`）。

```
Bundled ConPTY is unavailable; using the in-box conhost.
Kitty graphics and Sixel passthrough may be stripped on this Windows build.
```

**noctty 自身が「in-box conhost は kitty graphics と Sixel を剥ぐ」と明言していて、その対策として ConPTY v2 を同梱する機構を持っていました。**

仕組みは単純です。Microsoft が nuget で配布している ConPTY 再頒布物（`Microsoft.Windows.Console.ConPTY`、MIT、Windows Terminal の各リリースに同梱される署名済みパッケージ）の `conpty.dll` と `OpenConsole.exe` を、**exe と同じディレクトリに置くだけ**。あとは自動的に in-box conhost の代わりに使われます。設定オプションはありません。

リポジトリの `dist/windows/conpty-redist.json` に nupkg と展開後 2 ファイルの SHA256 がピンされているので、検証しながら取得できます。

```bash
# pin から URL とハッシュを読んで取得・検証する (抜粋)
url=$(jq -r '.nupkg.url' dist/windows/conpty-redist.json)
sha=$(jq -r '.nupkg.sha256' dist/windows/conpty-redist.json)
curl -fsSL -o package.nupkg "$url"
[ "$(sha256sum package.nupkg | cut -d' ' -f1)" = "$sha" ] || exit 1
```

配置して再起動したら、T3 は `ESC _ Gi=31;OK ESC \` を返し、画像も出るようになりました。**T1 も `?62;22;52c` に変わり、DA1 の横取りまで止まりました。**

:::message alert
**同梱で全部が解決するわけではありません**

再頒布物が効くのは **noctty 自身がホストするコンソールだけ**です。WSL シェルの中から `cmd.exe` を起動するような**ネストしたコンソールは System32 の in-box conhost に回される**ので、mangling が universal に消えるわけではありません。

また Microsoft はこの nuget パッケージのサードパーティ利用を**正式にはサポート表明していません**（[microsoft/terminal#15065](https://github.com/microsoft/terminal/issues/15065) が open のまま）。検証できるのは「first-party・検証済みプレフィックス・MIT・署名済みで Windows Terminal の各リリースに同梱されている」という事実までです。
:::

### 速度も別物になった

画像が出るようになったのは分かった。では描画性能はどうか。**同じバイナリで ConPTY バックエンドだけを入れ替えて A/B を取りました。**

条件を揃えるのが地味に大事なので書いておきます。

- 同一バイナリ・同一ウィンドウサイズ（140x37）・90 秒差で連続実行・各 3 回
- `NOCTTY_CONPTY=inbox` で in-box を強制
- **`--single-instance=false` を必ず付ける**（後述）

| 項目 | in-box conhost (v1) | **同梱 ConPTY (v2)** | 倍率 |
|---|---|---|---|
| **Plain scroll (50k 行)** | 0.065s | **0.023s** | **2.83x** |
| **Unicode/日本語 (10k 行)** | 0.176s | **0.123s** | **1.43x** |
| ANSI 16color (10k) | 0.106s | 0.104s | 1.02x |
| Wide chars (10k) | 0.101s | 0.107s | 0.94x |
| 256color (10k) | 0.109s | 0.107s | 1.02x |
| Cursor movement (5k) | 4.818s | 5.055s | 0.95x |

条件を変えた計測も足して統合すると（同梱 n=12 / in-box n=6）、影響を受ける 2 項目は**2 群の分布がまったく重なりません**（同梱の最大 0.026s < in-box の最小 0.062s）。ウィンドウの高さを 35 / 37 / 58 と変えても同梱側は 0.021〜0.026s で動きませんでした。

そして **Plain scroll の 0.023s は、vsock 直結の GhostInTheWSL とまったく同値**でした。「バッファへ書いて再レンダリング」をやめれば、そのぶんが丸ごと消えるわけです。

:::message
**A/B を再現するときの罠**

noctty は**シングルインスタンス**です。2 回目の起動は既存プロセスに argv を IPC 転送して自分は終了します。ConPTY の解決は `std.once` によるプロセス単位なので、`NOCTTY_CONPTY=inbox` を付けても**転送されると効きません**。

私はこれで一度、両方の窓が同梱 ConPTY で動いている状態のデータを取ってしまいました。検知方法は `tasklist` のプロセス数で、窓が 2 つあるのにプロセスが 1 つならこれです。`--single-instance=false` を付けてください。

ついでに `--window-width` / `--window-height` は**セル単位**で、かつ**両方指定しないと無視されます**（`src/Surface.zig` で `window_height <= 0 or window_width <= 0` なら早期 return）。
:::

### 実用上いちばん大事な話

**公式リリースには同梱 ConPTY が入っていますが、`zig build` は配置してくれません。**

自前ビルドを常用していると、気付かないまま in-box conhost で動きます。私はこれで数ヶ月間、noctty を「ConPTY 経由だから遅いターミナル」として誤って評価していました。

しかも**実行時のハッシュ検証はありません**（`LoadLibraryEx` してアーキテクチャを見て `GetProcAddress` するだけ）。古いファイルが残っていても警告なしで使われます。

どちらが有効かは **`noctty +version`** か diagnostic bundle のマニフェストで確認できます。自前ビルド派は `zig-out` を消したときと noctty 側が pin を上げたときに配置しなおす必要があるので、私は home-manager にヘルパーを書きました。

```nix
# hosts/wsl-gentoo.nix
# 取得元とハッシュは checkout 内の dist/windows/conpty-redist.json から読むので、
# noctty 側が pin を上げれば追随する。版をここに焼き込まない。
home.file.".local/bin/install-noctty-conpty" = {
  executable = true;
  text = ''
    # ... pin を jq で読み、nupkg と展開後 2 ファイルの SHA256 を検証して配置 ...
    if sha_is "$dest/conpty.dll" "$dll_sha" && sha_is "$dest/OpenConsole.exe" "$exe_sha"; then
      echo "OK: 配置済み (SHA256 一致)。何もしません"
      exit 0
    fi
  '';
};
```

## PR を 4 本出した話

noctty を評価する過程でバグを踏み、そのたびに PR を出しました。結果は 4 本ともマージ、外部コントリビュータとしてはマージ数 1 位になっていました。

| PR | 内容 | 提出→マージ |
|---|---|---|
| [#197](https://github.com/amanthanvi/noctty/pull/197) | URL 風文字列をパス解決に流さない | 33 時間 |
| [#198](https://github.com/amanthanvi/noctty/pull/198) | UIA で UI スレッドをレンダラロックでブロックしない | 23 時間 |
| [#209](https://github.com/amanthanvi/noctty/pull/209) | タブラベルが端末タイトルに追従するように | **1 時間 2 分** |
| [#215](https://github.com/amanthanvi/noctty/pull/215) | preedit のシェイパーセル追い付きループの範囲外アクセス | **1 時間 37 分** |

中身を簡単に紹介します。どれも「症状から見当をつけた原因」と「実際の原因」がずれていたのが面白いところでした。

### #197 — URL を Ctrl+Click するとプロセスが消える

WSL のシェルで URL を Ctrl+Click すると、ウィンドウごと消える。当初は「パス解決が assert で落ちている」と踏んでいました。

実際の真因は **`std.posix.faccessatW` の中にある `.OBJECT_NAME_INVALID => unreachable`** でした。`catch` を書いても拾えません。unreachable は Zig のパニックであってエラーではないからです。

さらにレビュー（greptile / coderabbit）で C0 制御文字と ADS（代替データストリーム）名の扱いを指摘され、**プローブを書いて実ファイルシステムで実測**したところ、当初のガードは 3 つの落ちる形をすり抜けていました。ここで自分の以前の説明（「トリガーは `?` である」）が誤りで、実際は**余分なコロン**だと判明しました。最終的に 27 パターンをクロス検証して書き直しています。

### #198 — 放置して戻ると 1 分以上フリーズする

UI スレッドが Win32 のメッセージディスパッチ経由で `windowProc` に再入し、アクセシビリティ（UIA）のスナップショット処理が `renderer_state.mutex` を**ブロッキング取得**したまま端末全体のテキストを走査していました。レンダラも termio も PTY も全員そこで詰まります。

修正は `lock()` → **`tryLock()`**。取れなければキャッシュ済みスナップショットを返して再試行タイマーを張り、UI スレッドは決してブロックしない。A/B で **78,168ms → 154ms** になりました。

再現手順を確立するのに一番時間がかかりました。直感に反して「速く叩くほど競合する」が**逆**で、高頻度ポーリングだと内部の `queryRecentlyActive()` が常に真になり、再入の源である同期メッセージが一度も走らないのです。1500ms 間隔が正解でした。

### #209 — タブのタイトルが最初のまま固まる

Claude Code を動かすと、GhostInTheWSL ではタブにプロンプトの要約が出るのに、noctty では `1: DESKTOP-XXXX:~` のまま変わらない。

原因は**呼び出しグラフが繋がっていない**ことでした。タブラベルを更新する `syncTabButtons()` の呼び出し元は `refreshChrome()` ただ 1 箇所なのに、OSC タイトル変更の経路はそこに到達しない。だからフォーカス変更など無関係なイベントが起きるまで固まる。

……というのが第一層で、**第二層がありました**。配線を足したらセッションログに `error.InvalidUtf8` が大量に出ていて、ラベルを縮める `compactHostLabel` が**バイト境界で切っていた**のです。日本語タイトルだとマルチバイト文字の途中で切れて不正な UTF-8 になり、`refreshChrome` 全体が途中で中断していました。つまり**元々壊れていて、私の変更がそれを表面化させた**形です。

### #215 — IME で変換中にウィンドウが消える

日本語入力の変換中に、ときどきウィンドウが無言で消える。WER にもダンプにも何も残らない。

stderr を捕捉しながら常用して、ようやく捕まえました。

```
thread 60552 panic: index out of bounds: index 5, len 5
  src\renderer\generic.zig:2565:32  in rebuildCells
```

preedit（変換中テキスト）の描画分岐にあるシェイパーセルの追い付きループが未ガードで、run が span より少ないセル数に shape されると末尾を踏み越える。**同じ関数の主経路には同じガードが既にありました。**

これは noctty 固有ではなく **upstream の Ghostty 本体にも残っているバグ**でした。GhostInTheWSL で出なかったのは、そちらが `ReleaseFast` ビルドで境界チェックが無効だっただけ、つまり**パニックせずに黙って UB になっていた**だけです。

なお Zig のパニックは `ExitProcess(3)` で終わるため SEH を上げません。だから WER にも `%LOCALAPPDATA%\CrashDumps` にも何も残らない。「無言終了」の正体はこれでした。

## AI 支援で PR を出すときに効いたこと

noctty には [`AI_POLICY.md`](https://github.com/amanthanvi/noctty/blob/main/AI_POLICY.md) があります。要約すると:

> AI 支援は許可する。ただし結果の責任は投稿者が負う。
> 意味のある AI 支援は PR で開示すること。提出するものはすべてレビューし理解すること。
>
> **その変更を説明できない、失敗をデバッグできない、AI に聞き直さずに設計を正当化できないなら、その変更はまだ提出できる状態ではない。**

最後の一文が実質的な合格ラインです。私の経験上、ここを満たすために効いたのは次の 3 つでした。

**1. 数値と再現手順を必ず添える**

「速くなりました」ではなく「78,168ms → 154ms、再現手順はこれ」。#198 はこれで通ったと思っています。逆に #197 の初版では「`openUrl` の UI スレッド同期呼び出しが 180〜430ms のブロックを起こす」と書いたのですが、**計測方法が間違っていた**（プローブを呼び出しごとに別プロセス起動していた）。測り直したら 5 分間の worst が 8.0ms で、実害を実証できませんでした。この項目は自分で取り下げています。

**2. 「直した」と「元々壊れていた」を分ける**

#209 の第二層がそうです。自分の変更が原因なのか、既存のバグを表面化させただけなのかを切り分けて書く。ここを曖昧にすると、レビュアーは差分の外側を見に行けません。

**3. レビューの指摘を鵜呑みにも門前払いにもしない**

coderabbit から「`max_len` は文字数として扱うべき」という指摘を受けました。もっともらしいのですが、その予算はピクセル幅から導出されているので、文字数で数えると CJK ラベルが確保領域を 2 倍はみ出します。**採用せず、理由を `file:line` 付きで返信**しました。本当に正しい解（表示幅ベースの予算）は別 PR に切る、と添えて。

一方 #197 の C0 制御文字と ADS の指摘は完全に正当だったので、プローブを書いて実測し、全面的に書き直しました。

### 私がやらかしたこと

正直に書いておくと、#209 で一度リグレッションを作りました。

タブラベル更新の配線を足すとき、既存の同期処理を `try` で伝播する経路に足したのです。noctty の Win32 メッセージループは `App.tick` を `try` で呼んでいるので、**エラーが 1 つ漏れるとメッセージループごと巻き戻ってプロセスが終了します**。既存の `refreshChrome` 呼び出し元が全部エラーをログに落として握り潰していたのは、まさにそのためでした。

教訓としては、**呼び出しを 1 つ足すとエラー集合が広がる**。既存の呼び出し元がどう握り潰しているかを先に見るべきでした。修正版ではログに落とすヘルパーを噛ませています。

## 導入

### インストール

Windows 10 1809 (build 17763) 以降、x64 か ARM64、OpenGL 4.3 以降の GPU ドライバが要ります。

```powershell
scoop bucket add noctty https://github.com/amanthanvi/scoop-noctty
scoop install noctty/noctty
```

インストーラとポータブル ZIP もリリースページにあります（WinGet は準備中）。**別アプリとしてインストールされる**ので、今のターミナルを残したまま試せます。

### 設定

Ghostty と同じ設定体系です。初回起動時に `%LOCALAPPDATA%\noctty\config.ghostty` にテンプレートが書かれます。私の WSL 用設定はこうです。

```ini
command = direct:wsl.exe -d Gentoo-systemd --cd ~
copy-on-select = clipboard
font-family = UDEV Gothic JPDOC
font-family = UDEV Gothic NF
font-family = Segoe UI Symbol
font-size = 13
keybind = ctrl+l=next_tab
keybind = ctrl+h=previous_tab
theme = iTerm2 Solarized Light
window-inherit-working-directory = false
tab-inherit-working-directory = false
split-inherit-working-directory = false
```

`direct:` プレフィックスは `/bin/sh -c` ラップを回避するためのもので、Windows には sh が無いので WSL 直起動では必須です。`Ctrl+Shift+,` で設定をライブリロードできます。

### `*-inherit-working-directory` を切っている理由

これは知らないとハマります。noctty は WSL の直起動コマンドを内部で書き換えます。ユーザーが書いた `--cd ~` を**無条件に除去して**、解決済みの cwd を `--cd` として注入し直すのです。

解決順は「継承 / 明示 cwd > `working-directory = home`」なので、既定（`true`）のままだと `--cd ~` は常に無視されます。しかも初回タブは cwd 未確定なので**起動プロセスの Windows 側 cwd** をそのまま採用し、新規タブがそれを継承します。`zig-out\bin` から起動していると、新しいタブが `/mnt/c/.../zig-out/bin` で開きます。

そもそも WSL 側には shell integration が届かず OSC 7 が来ないので、cwd 継承は構造的に正しく機能しません。3 つとも切って `working-directory = home` を使わせるのが正解でした。

なお PowerShell や cmd を使う場合はこの限りではなく、noctty は **PowerShell への自動インジェクション**と **cmd.exe のプロンプト / cwd マーク**（Clink が必要）を持っているので、OSC 7 も OSC 133 も飛びます。

### 同梱 ConPTY

繰り返しになりますが、ここが一番大事です。

- **公式リリースを使う**なら何もしなくて構いません
- **自前ビルド**なら `conpty.dll` と `OpenConsole.exe` を exe の隣に配置してください
- 確認は `noctty +version`

## まとめ

| | 状況 |
|---|---|
| Windows 統合 | 既定ターミナル登録・エクスプローラメニュー・ジャンプリスト・統合タイトルバー |
| ターミナル機能 | セッション復元・名前付きレイアウト・Quick select / copy mode・パレット・ペインへの drag-to-split |
| 省電力 | 背面 FPS 制限、最小化 / クローク時は present しない |
| プライバシー | テレメトリなし。ダンプはローカル。更新チェックは切れる |
| バルク描画 | 同梱 ConPTY で vsock 直結と同値（Plain scroll 0.023s） |
| 画像プロトコル | 同梱 ConPTY で通る |
| コントリビュート | AI 支援が明文で歓迎。PR は最短 1 時間でマージ |
| 落とし穴 | **`zig build` は同梱 ConPTY を配置しない** |

「ConPTY 経由だから遅い」という定説は、**層の特定としては正しかったが、その層が差し替え可能である**という点が抜けていました。ConPTY には v1 と v2 があり、v2 は元の VT バイトをそのまま流します。私はこれを知らずに数ヶ月、判断を誤っていました。

同じ思い込みをしている人がいれば、まず `noctty +version` でどちらの ConPTY を掴んでいるか確認してみてください。

そしてもし noctty を使っていてバグを踏んだら、直して投げるのが早いです。数時間で返ってきます。

## 続編

機能単位で Windows Terminal と比較した続編を書きました。

- [noctty と Windows Terminal を比べる — どちらを選ぶかは「GPU とスクリーンリーダー」で決まる](https://zenn.dev/nanasess/articles/noctty-vs-windows-terminal)

noctty を**選んではいけない 3 つの条件**（OpenGL 4.3、スクリーンリーダー、組織配布）と、noctty のベンチマークハーネスに PresentMon アダプタを足して非公式に測った性能を載せています。

## 参考リンク

- [noctty (amanthanvi/noctty)](https://github.com/amanthanvi/noctty)
- [noctty の AI usage policy](https://github.com/amanthanvi/noctty/blob/main/AI_POLICY.md)
- [Windows capability matrix](https://github.com/amanthanvi/noctty/blob/main/docs/windows-capability-matrix.md) — upstream Ghostty との機能単位の比較
- [Windows VT conformance](https://github.com/amanthanvi/noctty/blob/main/docs/windows-vt-conformance.md) — ConPTY 世代と mangling カタログ
- [Ghostty](https://ghostty.org/)
- [GhostInTheWSL (Codavo/ghostinthewsl)](https://github.com/Codavo/ghostinthewsl) — vsock 直結の WSL 専用フォーク
- [microsoft/terminal#17510](https://github.com/microsoft/terminal/pull/17510) — ConPTY v1 / v2 の境界
- [Windows Terminal Preview 1.22 リリースノート](https://devblogs.microsoft.com/commandline/windows-terminal-preview-1-22-release/) — ConPTY 書き直しと直接 VT 転送
- [microsoft/terminal#15065](https://github.com/microsoft/terminal/issues/15065) — ConPTY nuget パッケージのサードパーティ利用（open）
- [kitty graphics protocol](https://sw.kovidgoyal.net/kitty/graphics-protocol/)
