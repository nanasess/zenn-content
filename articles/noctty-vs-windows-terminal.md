---
title: "noctty と Windows Terminal を比べる — どちらを選ぶかは「GPU とスクリーンリーダー」で決まる"
emoji: "⚖️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["noctty", "windowsterminal", "terminal", "windows", "wsl"]
published: false
---

:::message
この記事は Claude Code で執筆しています。noctty と Windows Terminal の一次資料の調査も [Claude Code](https://claude.com/claude-code) が実施しました。
:::

以前、[WSL 用ターミナルは noctty が良いぞ](https://zenn.dev/nanasess/articles/noctty-wsl-terminal-contribution) という記事を書きました。今回はその続きとして、**Windows Terminal と noctty を機能単位で比較**します。

前回は「noctty が良い」という話でしたが、今回は逆側も書きます。**noctty を選んではいけないケースが明確に 3 つある**ためです。

## 結論

先に要点だけ。

**noctty を選べないケース（この 3 つのどれかに当たるなら Windows Terminal 一択）**

1. **RDP・VM・古い GPU が主戦場** — noctty は OpenGL 4.3 が絶対条件で、ソフトウェアフォールバックが存在しません。下回ると起動を中止します
2. **スクリーンリーダーを使う** — noctty 側のドキュメントが自ら「Narrator・NVDA・JAWS に依存しているなら Windows Terminal に留まれ」と書いています
3. **組織で配布したい** — 自己署名証明書のため SmartScreen 警告が消えません。WinGet も未公開です

**noctty を選ぶ理由が立つケース**

- Kitty グラフィックスプロトコル（画像表示）を使う。**これは Windows Terminal に無い**
- 分割の undo/redo、名前付きレイアウト、Quick select、コピーモードといったキーボード中心の操作が欲しい
- テレメトリを一切通したくない
- Ghostty の設定文法とテーマ資産をそのまま使いたい

**そして重要なこと: 両者は排他ではありません。** noctty のドキュメント自身が「Windows Terminal を残したまま試せ」と書いていますし、後述するように **noctty は既定ターミナル機能のために Windows Terminal を必要とします**。

## 比較の土台

この記事の根拠を先に明示します。ターミナルの比較記事は「体感」で書かれがちですが、それだと読者が判断できないので。

| 区分 | 根拠 |
|---|---|
| **noctty 側** | リポジトリの [`docs/`](https://github.com/amanthanvi/noctty/tree/main/docs) 配下 27 ファイルを通読。該当箇所のファイル名を都度示します |
| **Windows Terminal 側** | 公式リリースノートと microsoft/terminal の Issue を確認。URL を都度示します |
| **性能** | **比較しません**（理由は後述） |

なお noctty のドキュメントは、この種の比較記事にとって珍しく使いやすい資料です。[`docs/windows-capability-matrix.md`](https://github.com/amanthanvi/noctty/blob/main/docs/windows-capability-matrix.md) が upstream Ghostty のドキュメント項目を 1 行ずつ `supported` / `partial` / `windows-specific` に分類していて、`partial` の内訳が具体的に書かれています。前回の記事でも触れましたが、比較の土台として引用できるレベルで正直です。

## ターミナルコアの差

### 画像プロトコル: ここが最大の差

| | noctty | Windows Terminal |
|---|---|---|
| **Kitty グラフィックスプロトコル** | 対応 | **非対応**（[#8389](https://github.com/microsoft/terminal/issues/8389) が 2020 年 11 月から open。同じ要望の [#17309](https://github.com/microsoft/terminal/issues/17309) は 2024 年に重複としてクローズ） |
| **Sixel** | 対応 | **1.22 で対応**（[リリースノート](https://devblogs.microsoft.com/commandline/windows-terminal-preview-1-22-release/)） |

Sixel は Windows Terminal 1.22 で入ったので、もう差ではありません。`img2sixel` や `chafa` を通す運用なら両方で動きます — ただしこれは後述する **ConPTY v2 を経由する場合**の話です。Windows Terminal 1.22 以降か noctty の同梱 ConPTY v2 なら通りますが、v1 や in-box conhost に落ちる経路では DCS ごと剥がされます。

**残る差は Kitty グラフィックスプロトコル**です。`timg`、`yazi` のプレビュー、Neovim の画像プラグインなど、Kitty プロトコル前提のツールを使うなら noctty 側に理由が立ちます。

### キーボードプロトコル: ここは差が縮まった

**この記事を書く過程でいちばん認識を改めた点です。**

私は当初「Kitty キーボードプロトコルは noctty の優位点」だと思っていました。**違いました。Windows Terminal は 1.25 でこれに対応しています**（[microsoft/terminal#11509](https://github.com/microsoft/terminal/issues/11509)）。

したがって、`Ctrl+,` と `Ctrl+.` の区別、キーリリースイベント、左右修飾キーの識別といった、Neovim や Helix で効いてくる部分は**両方で使えます**。

ただし Windows Terminal 側には注意点があります。

- **AltGr との組み合わせが入力できない**（1.25.622.0 で報告。[#20361](https://github.com/microsoft/terminal/pull/20361) に「AltGr 合成文字が合成結果ではなく `alt+<key>` の CSI u として符号化される」と現象が説明されていますが、この修正 PR はマージされずクローズされています）
- **F13〜F20 が Kitty のシーケンスではなく従来のものになる**（[#20243](https://github.com/microsoft/terminal/issues/20243)。2026 年 9 月に `Resolution-By-Design` でクローズされ、不具合ではなく仕様という整理になりました）

noctty 側は AltGr の扱いを明示的に設計しています（[`docs/windows.md`](https://github.com/amanthanvi/noctty/blob/main/docs/windows.md) の Keyboard 節）。Windows が AltGr 押下時に合成する「左 Ctrl + 右 Alt」を破棄し、レイアウトの文字を出す。AltGr マッピングを持つレイアウトでのみこの折り畳みを適用する、という条件まで書かれています。日本語キーボードでは AltGr が無いので影響しませんが、欧州系レイアウトを使う人には効きます。

### ConPTY: 「ConPTY だから遅い」はもう古い

前回の記事で詳しく書いたので要点だけ。ConPTY には世代があり、境界は [microsoft/terminal#17510](https://github.com/microsoft/terminal/pull/17510) です。

- **v1**（Windows Terminal 1.21 以前、および検証されたすべての in-box conhost）: 子の VT をパースしてスナップショットを再レンダリングするので、未知のシーケンスが落ちる
- **v2**（Windows Terminal 1.22 以降と再頒布物）: 元の VT バイトを別途そのままパイプへ書く

**つまり v2 の恩恵は Windows Terminal ユーザーも受けています。** noctty の優位は「v2 を同梱していて、in-box conhost に落ちたときの劣化を回避できる」という点に絞られます。

なお `docs/windows-vt-conformance.md` の [Measured child-to-master differential](https://github.com/amanthanvi/noctty/blob/main/docs/windows-vt-conformance.md#measured-child-to-master-differential) 節には同一マシン（Windows `10.0.26200.0`、in-box conhost `10.0.26100.1`、同梱 `conpty.dll` `1.24.260710001`）での実測が載っていて、in-box (v1) では Kitty graphics APC と Sixel DCS が**完全に消失**、バンドル版 (v2) では**バイト単位で一致**と記録されています。printable マーカーは両方で生存しているので、パイプが空だったのではなく確かにシーケンスが剥がされている、という書き方まで含めて丁寧です。しかも「この測定が示すのは伝送の生存だけで、Kitty のピクセル描画のギャップを埋めるものではない」という但し書きまで付いています。

そして注意書きも重要です。**in-box conhost の世代を OS のビルド番号から推測してはいけない**。v2 を含む最初の Windows ビルドは特定できていない、と明記されています。

## noctty を選べない 3 つのケース

ここが本題です。

### 1. OpenGL 4.3 が絶対条件

[`docs/windows.md`](https://github.com/amanthanvi/noctty/blob/main/docs/windows.md) の GPU floor 節に書かれています。

> noctty は WGL 経由の OpenGL 4.3 以降を必要とし、ソフトウェア・DirectX・ANGLE のいずれのフォールバックレンダラも持たない。したがって、有効な OpenGL 実装がその下限を下回るときは起動できない。

下回った場合、**ウィンドウを出す前に起動を中止**し、要求バージョンと検出バージョン、レンダラ・ベンダ文字列を並べたダイアログを出します。該当しやすい環境も明示されています。

- ソフトウェア GL に落ちる **RDP セッション**（多くは `GDI Generic` の OpenGL 1.1）
- 3D アクセラレーションやゲスト GPU ドライバのない **VM**
- 4.3 に届かない**古い統合 GPU**

Windows Terminal はこの制約を持ちません。**リモートデスクトップや仮想マシンで作業するなら、この一点で noctty は候補から外れます。**

ちなみに `LoadLibrary failed with error 126` で落ちるケースも文書化されていて、AMD + NVIDIA のハイブリッド GPU ノートで WGL が DriverStore から AMD の OpenGL ICD を読む際に多い、ドライバは AMD → NVIDIA の順で入れ直せ、と具体的です。

### 2. アクセシビリティが未成熟

[`docs/accessibility-matrix.md`](https://github.com/amanthanvi/noctty/blob/main/docs/accessibility-matrix.md) の記述が、正直すぎるほど正直です。

- **リリースビルドに対してスクリーンリーダーを計測した実績がゼロ**
- Narrator 未計測、JAWS 未計測（商用ライセンス未所持のため）
- NVDA のみ部分計測。結果は**まちまち**

NVDA の結果の内訳がまた具体的です。

| 読める | 読めない |
|---|---|
| ターミナルテキスト、スクロールバー、ライブリージョン、ホストバナー | **タブ項目・検索フラグのトグル・ドッキング検索の入力欄が role / state / name をアナウンスしない** |

原因も特定されています。NVDA はカスタムウィンドウクラスでは noctty の UIA プロバイダを読むが、**標準 Win32 の `Button` / `Edit` クラスでは MSAA にフォールバックしてしまう**。キャプションボタンに至っては HWND を持たないフラグメントなので、**NVDA のオブジェクトナビゲーションが構造上到達できない**（マウスを乗せたときだけ読む）。

そして [`docs/migrate-from-windows-terminal.md`](https://github.com/amanthanvi/noctty/blob/main/docs/migrate-from-windows-terminal.md) の「Honest gaps」節が、結論をこう書いています。

> Narrator、NVDA、JAWS に今日依存しているなら、Windows Terminal に留まり、[マトリクス](https://github.com/amanthanvi/noctty/blob/main/docs/accessibility-matrix.md)を追ってほしい。

自分のプロダクトの移行ガイドに「移行するな」と書けるのは、なかなかできることではないと思います。

### 3. 署名と配布

- **自己署名証明書**（`CN=winghostty Local Dev Signing`）のため、**SmartScreen の警告は時間が経っても消えません**（[`docs/getting-started.md`](https://github.com/amanthanvi/noctty/blob/main/docs/getting-started.md)）
- WinGet はブートストラップ待ち、Chocolatey は未公開。実質 Scoop か手動ダウンロード

[ADR 0006](https://github.com/amanthanvi/noctty/blob/main/docs/adr/0006-code-signing-certificate-decision.md) で Azure Artifact Signing への移行が決定済みですが、Microsoft 自身のドキュメントを引いて「**どの選択肢も初日に警告を消さない**」「EV 証明書によるバイパスは 2024 年に廃止された」と書かれています。移行のオーナー前提条件（Azure の個人 ID 検証、1〜20 営業日、短縮不可、書類提出 3 回まで）も**まだ完了していません**。

代わりに noctty がやっているのは**公開鍵のピン留め**です。更新の署名者の SPKI SHA-256 をアプリにコンパイル時に焼き込み、他の鍵で署名されたものは署名として妥当でも拒否する。[ADR 0005](https://github.com/amanthanvi/noctty/blob/main/docs/adr/0005-pin-updater-publisher-public-keys.md) にその設計が、[`docs/signing-rotation-runbook.md`](https://github.com/amanthanvi/noctty/blob/main/docs/signing-rotation-runbook.md) に鍵ローテーションの手順が書かれています。この runbook は各ステップに「**このステップを飛ばすとどうなるか**」が併記されていて、運用文書として良くできています。

とはいえ、**組織のソフトウェア配布ポリシーを通すのは難しい**でしょう。個人利用向けの選択肢です。

## Windows Terminal にあって noctty に無いもの

上の 3 つ以外にも、機能単位でのギャップがあります。

| ギャップ | 詳細 |
|---|---|
| **ユーザー定義プロファイルが無い** | noctty のプロファイルは**インストール済みシェルの自動検出**で、ユーザーが書くプロファイルスキーマが存在しません。Windows Terminal の per-profile な色・フォント・アイコン・開始ディレクトリ・`elevate` フラグに相当するものがない |
| **ウィンドウ間のタブ移動ができない** | 同一ウィンドウ内の並べ替えとドラッグ分割のみ。クロスウィンドウの OLE 転送は未実装 |
| **既定ターミナルとして完結できない** | パッケージ ID が無いため Windows 設定のピッカーに出せず、`IConsoleHandoff` も未実装 |
| **モダンコンテキストメニューに出ない** | Windows 11 では「その他のオプションを表示」配下のクラシック動詞のみ |
| **`settings.json` インポータが無い** | 存在せず、**作る計画も無い**と明言されています |
| **リンクプレビューが描画されない** | マッチ・ホバー・オープンは動くがツールチップ未実装 |
| **ステータスバーが無い** | `Host.statusBarHeight()` が 0 を返す。出荷しない方針 |
| **Acrylic / Mica そのものは移らない** | DWM の tabbed backdrop を要求する形で、ぼかし半径はオン / オフ扱い。Windows 10 と 11 21H2 では受理されるが何も起きない |

プロファイルの件は、`docs/windows.md` の昇格ウィンドウの節に経緯が書かれていて面白いです。

> ロードマップは当初 per-profile の `run elevated` フラグを求めた。noctty の Windows プロファイルはユーザーが書くプロファイルスキーマではなくインストール済みシェルから検出されるので、設定すべきプロファイル属性が存在しない。per-profile の等価物は `new_window_elevated:<profile-key>` のバインドである。

「実装しなかった」ではなく「アーキテクチャ上そこに置く場所が無い」という説明になっています。

### 既定ターミナルの話は、もう少し込み入っています

noctty の `+register-default-terminal` は実験的機能ですが、**Windows Terminal 1.24 以降の OpenConsole がコンソール側として選択されたままであることを要求します**。noctty はコンソールの半分を自前で実装していないからです。

つまり **noctty を既定ターミナルとして使うには、Windows Terminal を消せません**。「乗り換え」ではなく「共存」が前提の設計です。

さらに `docs/windows.md` には、登録のセキュリティ上の性質が正直に書かれています。

> 登録すると同一ユーザーの同一整合性レベルで動く任意のローカルプロセスが noctty のハンドオフクラスを直接アクティブ化し、自分が制御するパイプとウィンドウタイトルで `EstablishPtyHandoff` を呼べる。結果は、そのプロセスの内容を描画する本物の noctty ウィンドウであり、ユーザーのキー入力がそこへ流れる。たとえば `Administrator: Windows PowerShell` というタイトルのウィンドウ、というフィッシングの形になる。

そのうえで「特権境界は越えていない（そのプロセスは既にユーザーとして動いており、他の方法でもターミナルを騙れる）」と続きます。**実験的機能を勧めるときに、その攻撃形状を先に書く**姿勢は信頼できます。

## noctty にあって Windows Terminal に無いもの

逆側です。

### キーボード中心の操作

| 機能 | 内容 |
|---|---|
| **構造的 undo/redo** | 分割の作成、単一タブのクローズ、ドラッグ分割の部分木転送を undo できる。タブの正確な形・ペイン・レイアウト・比率・フォーカスまで復元される |
| **Quick select** | `Ctrl+Shift+Space` で画面上の URL・パス・git SHA・IP・UUID にラベルを振り、キーボードで選択 |
| **コピーモード** | `Ctrl+Shift+X` で vi 風のモーダル選択とスクロールバック移動 |
| **名前付きレイアウト** | 1 ウィンドウのタブ / 分割 / プロファイル / cwd / タイトルの形を保存して再現 |
| **ユニバーサルパレット** | アクション・タブ・ペイン・プロファイル・レイアウト・テーマ・設定・最近のコマンドを 1 つのファジー検索面に統合。プレフィックス（`>` `@` `/` `~` `:` `%` `!` `^` `?`）でカテゴリ絞り込み |
| **SSH ホスト検出** | `~/.ssh/config` の具体的なエイリアスをプロファイルピッカーとパレットに並べ、`ssh.exe` で起動 |

Quick select の設計が細かくて良いです。Ctrl を押しながら確定すると URL を開けますが、**`file:` URL はラベル付けもコピーもできるが Ctrl では開かない**。理由も書かれています — システムのハンドラが `file:///C:/….exe` をターミナル出力から直接実行してしまうから。

さらに、確定時にターゲットをライブ画面と**再照合**します。オーバーレイが出ている間に走っているプログラムがその領域を書き換えていたら、そのターゲットは無視される。「画面に出ていたものにしか作用しない」という保証です。

### 構造化された CLI 自動化

[`docs/automation.md`](https://github.com/amanthanvi/noctty/blob/main/docs/automation.md) の面です。

```powershell
$state = noctty +list-windows --class=work | ConvertFrom-Json
$pane = $state.windows[0].tabs[0].panes[0]
noctty +focus --class=work --surface-id=$($pane.surface_id)
noctty +send-text --class=work --surface-id=$($pane.surface_id) 'git status'
```

`noctty.windows.v3` というバージョン付き JSON スキーマで、ウィンドウ / タブ / ペインのメタデータが返ります。終了コード 0〜5 も安定契約です。

そのうえで、**JSON に含めないものが明示**されています。

> ターミナルのグリッドテキスト、スクロールバック、選択範囲、クリップボードの内容、保留中のシェル入力は、ペイロードに一切含まれない。

ペイン単位の子 PID も意図的に出していません（アプリスレッドから取得できないうえ、Windows では直接の ConPTY の子がラッパであることが多いため）。

`+send-text` の制約も徹底していて、**すべての Unicode `Cc` 制御文字を拒否**します。つまり **Enter を送れない = コマンドを実行させられない**。「印字可能なテキストでも TUI や確認プロンプトには影響しうる」という限界まで書いたうえで、「生モードやバイパスは存在しない」と締めています。

### プライバシー

| | noctty | Windows Terminal |
|---|---|---|
| **テレメトリ** | **なし**。アップロードするコードパスがリポジトリに存在しない | Windows の診断データ設定に従う。アプリ独自の opt-out は無く、OS 側の診断データ設定で制御する（文書化を求めた [#6118](https://github.com/microsoft/terminal/issues/6118) は 2020 年に完了扱いでクローズされ、ドキュメントリポジトリへ移管されました） |
| **外向き通信** | 更新チェックのみ（24 時間に 1 回、`auto-update = off` で停止） | — |
| **クラッシュダンプ** | `%LOCALAPPDATA%\noctty\crash` にローカル固定 | — |

Windows Terminal 側は「基本的な Windows テレメトリ設定なら Terminal から Microsoft へ送られるものは無い」という説明がなされています。**「テレメトリがある / ない」の二択で語るのは不正確**で、OS 設定に紐づく、というのが実情のようです。

noctty 側は [ADR 0004](https://github.com/amanthanvi/noctty/blob/main/docs/adr/0004-keep-diagnostics-local-and-explicit.md) で「診断はローカル生成、明示操作でのみエクスポート、ターミナル内容・クリップボード・環境・コマンドライン・cwd・生 config・ダンプは既定で除外」と決めています。`+diagnostic-bundle` も既定でそれらを落とします。

## 性能を比較しない理由

「どっちが速いの」が知りたいところだと思いますが、**この記事では比較しません**。

理由は、noctty 自身が比較を公開していないからです。[`docs/windows-benchmark-methodology.md`](https://github.com/amanthanvi/noctty/blob/main/docs/windows-benchmark-methodology.md) にこう書かれています。

> ハーネスは Alacritty、Windows Terminal、Tabby、Wave を検出するが、生産側だけのタイミングをターミナルのスループットとして報告することはしない。競合のメトリクスは、そのアダプタが、ターミナルのレンダリング計装か検証済みの PresentMon ETW キャプチャのいずれかによって、安定したプロセス / ウィンドウの所有権のもとで**同じ因果的エンドポイントを証明するまで `not-supported`** のままである。

つまり「同じ地点を測っていると証明できないので数字を出さない」という立場です。ベースラインマシンに Alacritty と Windows Terminal が入っていたことまで記録したうえで、**この構成だけから競合の性能結果は公開しない**と書いています。

この文書は他にも読みどころがあります。

- **性能目標を満たせていない数値を、調整せずそのまま公開している**。ペインあたりメモリ 32.14 MB（上限 20 MB）、コールドスタートは中央値 297 ms で上限内・p95 313 ms で上限超過（上限 300 ms）
- **CI のしきい値は中央値ではなく「観測された最遅の実行」の下**に置かれている。ウォームアップが無いので 1 回目が大きく下振れするから
- しきい値はすべて `passed: null` の inactive で、**ゲートを黙って通すことができない**設計
- アイドル計測で見つかったバグの記録が生々しい。**Num Lock がオンであるだけで、マウス移動のたびに同一フレームが present されていた**（修飾キーの比較で、生の値と正規化済みの値を突き合わせていたのが原因）

自分のプロダクトの数字を、性能目標を満たせていない状態のまま出せるかというと、なかなかできないと思います。

## 使い分けの指針

### Windows Terminal を選ぶべき

- RDP・VM・古い GPU が主な作業環境
- スクリーンリーダーを使う
- 既定ターミナルとして完全に置き換えたい
- per-profile の細かいカスタマイズ（色・アイコン・昇格フラグ）が要る
- 組織の配布・更新経路として Store / WinGet / MSIX が必要
- 単独メンテナのプロジェクトへの依存を業務上受け入れられない

### noctty を選ぶ理由が立つ

- ローカルの物理マシンで、OpenGL 4.3 以上の GPU がある
- **Kitty グラフィックスプロトコル**を使うツールがある
- 分割の undo/redo、名前付きレイアウト、Quick select、コピーモードが欲しい
- テレメトリを一切通したくない
- CLI からの構造化された自動化が要る
- Ghostty の設定文法とテーマ資産をそのまま使いたい

### 現実的な移行手順

`docs/migrate-from-windows-terminal.md` が勧めている手順が、そのまま最も安全です。

1. **両方入れる**（併存し、Windows Terminal の `settings.json` は読まれも変更されもしない）
2. **既定ターミナルは Windows Terminal のままにして評価する**
3. 実際のセッションを 1 つずつ通す — 新規タブと分割、プロンプトの開始ディレクトリ、複数行のコピー & ペースト、全画面 TUI、依存している SSH ワークフロー、そして再起動後の復元
4. 納得してから習慣を移す

設定の対応表も用意されています。ハマりどころを 2 つだけ挙げておくと、

- `opacity` → `background-opacity` は **100 で割る**（`50` → `0.5`）
- `historySize` → `scrollback-limit` は **単位が違う**。Windows Terminal は行数、noctty は**バイト数**（既定 10 MB）

## まとめ

- **Sixel とキーボードプロトコルの差は、もう埋まっている**。Windows Terminal 1.22 で Sixel、1.25 で Kitty キーボードプロトコル
- **残る機能差は Kitty グラフィックスプロトコル**と、Windows ネイティブ UI の作り込み（undo/redo、名前付きレイアウト、Quick select、コピーモード、構造化自動化）
- **noctty を選べない条件は明快**。OpenGL 4.3、スクリーンリーダー、組織配布のどれかに当たるなら Windows Terminal
- **両者は排他ではない**。noctty の既定ターミナル機能はむしろ Windows Terminal を必要とする

個人的には、noctty のいちばんの価値は機能そのものより **ドキュメントの正直さ**だと思っています。「対応しています」で濁さず、どこまで動いてどこから動かないか、何を計測して何を計測していないかが書いてある。性能目標を満たせていない数字をそのまま出し、自分の移行ガイドに「あなたは移行するな」と書ける。

比較記事を書くときにこれほど引用しやすい相手も珍しく、この記事はほぼその副産物です。

## 参考リンク

### noctty

- [noctty リポジトリ](https://github.com/amanthanvi/noctty)
- [`docs/windows-capability-matrix.md`](https://github.com/amanthanvi/noctty/blob/main/docs/windows-capability-matrix.md) — upstream Ghostty との対応表
- [`docs/status.md`](https://github.com/amanthanvi/noctty/blob/main/docs/status.md) — 動くもの / 実験的なもの / 対象外
- [`docs/windows.md`](https://github.com/amanthanvi/noctty/blob/main/docs/windows.md) — Windows 挙動リファレンス
- [`docs/windows-vt-conformance.md`](https://github.com/amanthanvi/noctty/blob/main/docs/windows-vt-conformance.md) — ConPTY 世代とマングリングカタログ
- [`docs/accessibility-matrix.md`](https://github.com/amanthanvi/noctty/blob/main/docs/accessibility-matrix.md) — スクリーンリーダー実測記録
- [`docs/automation.md`](https://github.com/amanthanvi/noctty/blob/main/docs/automation.md) — CLI 自動化
- [`docs/windows-benchmark-methodology.md`](https://github.com/amanthanvi/noctty/blob/main/docs/windows-benchmark-methodology.md) — ベンチマーク方法論
- [`docs/migrate-from-windows-terminal.md`](https://github.com/amanthanvi/noctty/blob/main/docs/migrate-from-windows-terminal.md) — 移行ガイド

### Windows Terminal

- [Windows Terminal Preview 1.22 Release](https://devblogs.microsoft.com/commandline/windows-terminal-preview-1-22-release/) — Sixel、書記素クラスタ、ConPTY 書き直し
- [microsoft/terminal#11509](https://github.com/microsoft/terminal/issues/11509) — Kitty キーボードプロトコル対応
- [microsoft/terminal#8389](https://github.com/microsoft/terminal/issues/8389) / [#17309](https://github.com/microsoft/terminal/issues/17309) — Kitty グラフィックスプロトコル（未対応）
- [microsoft/terminal#20243](https://github.com/microsoft/terminal/issues/20243) — F13〜F20 の Kitty シーケンス（`Resolution-By-Design` でクローズ）
- [microsoft/terminal#20361](https://github.com/microsoft/terminal/pull/20361) — AltGr 合成文字が KKP 下で落ちる（修正 PR は未マージ）
- [microsoft/terminal#17510](https://github.com/microsoft/terminal/pull/17510) — ConPTY v1 / v2 の境界
- [microsoft/terminal#6118](https://github.com/microsoft/terminal/issues/6118) — テレメトリの文書化要望（2020 年にクローズ、docs リポジトリへ移管）

### 関連記事

- [WSL 用ターミナルは noctty が良いぞ — PR を 4 本マージしてもらって分かったこと](https://zenn.dev/nanasess/articles/noctty-wsl-terminal-contribution)
