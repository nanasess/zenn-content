---
title: "脆弱性対応勉強会Expansion 第05回(OWASP ZAP&EC-CUBE)発表資料"
emoji: "🐝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ECCUBE", "OWASPZAP", "Security"]
published: true
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://source.unsplash.com/collection/94734566/1920x1080
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# persist drawings in exports and build
drawings:
  persist: false
---

2021年11月19日にオンライン開催した、脆弱性対応勉強会Expansion 第05回の発表資料をまとめた記事です。
*箇条書きで見づらいかもしれませんがご容赦ください*

https://twitter.com/hogehuga/status/1453695774550679556

[当日の録画はYoutubeにてご覧ください](https://www.youtube.com/watch?v=0CSzoz7B-mY)(2時間くらいあります)



# 自己紹介

- 名前: 大河内健太郎(@nanasess) 年齢: 44才
- 出身: 愛知県西尾市一色町
- 在住: 大阪市(四天王寺の近く)
- 前職:寿司屋の板前
- 資格: 調理師・ふぐ処理師
- EC-CUBE コミッター・公式エバンジェリスト・名古屋ユーザーグループ
- Emacs23-24 のアイコンを作った人です
- 最近のマイブーム: 寿司パ

---

# 本日のアジェンダ


*質問などありましたら [Twitter で #vulnstudy をつけてツイート](https://twitter.com/search?q=%23vulnstudy)お願いします！*

## EC-CUBEにおけるOWASP ZAPの侵入テストを自動化する試み

OWASP ZAPの侵入テスト自動化を試行錯誤しているお話。
大変手さぐりでやっていますので、間違ったことなどありましたら遠慮なくご指摘ください。

## EC-CUBEを安全に構築するための対策

EC-CUBEを安全に運用するために大切なこと。


# EC-CUBE のご紹介

- https://www.ec-cube.net
- オーンソースのEC構築パッケージ
- クラウド版もあり
- 最新バージョンは 4.1.0/2.17.2
- 構築事例はこちら(https://www.ec-cube.net/product/cases/backnumber.php)


# EC-CUBE における脆弱性対応

- セキュリティ対策について(https://www.ec-cube.net/info/security/)
- 脆弱性リスト(https://www.ec-cube.net/info/weakness/)


# EC-CUBE Penetration Testing with OWASP ZAP

- リリース前に OWASP ZAP で侵入テストを実施
  - VADDY(https://vaddy.net/ja/) でもテストをしている
- docker-compose コマンドで設定済みの OWASP ZAP を起動できる
- 誰でも実施できるようにドキュメントも作成しているので、ぜひお試しください(https://doc4.ec-cube.net/penetration-testing)


# ペネトレーションテストの課題

- OWASP ZAP を使用したペネトレーションテストを自動化する手段として、 [OWASP ZAP Full Scan
 ](https://github.com/marketplace/actions/owasp-zap-full-scan) がある
- しかし、以下のようなEC-CUBE特有の仕様で OWASP ZAP の自動スキャンでは十分にテストできない
  - 日本語入力が必須となっているフォーム
  - 特殊な画面遷移パターンがある
- [テストが止まらないようにするパッチ](https://doc4.ec-cube.net/penetration-testing/testing/apply_patch)を当てる必要がある
- 400以上のエンドポイントを毎回手動でテストするのは大変な労力がかかる
  - GitHub Actions で自動化したい

# 日本語入力が必須となっている入力フォ–ムがある

- 会員登録フォームのカナ氏名など、日本語入力が必須となっている
- OWASP ZAP の自動スキャンは日本語入力に対応していない
  - 入力画面から確認画面の遷移で入力エラーとなってしまい、テストが不十分で終わってしまう

**マニュアルスキャンで、日本語入力のリクエストを送信しておく必要がある**


# 特殊な画面遷移パターンがある

- 入力画面→確認画面→完了画面の遷移で、 URL を変えずに POST パラメータで遷移する機能がある
  - `mode=confirm`, `mode=complete` で遷移している
- OWASP ZAP は、同一キーの POST パラメータのリクエストを1つのページとして認識する
  - mode で画面遷移するページはテストが不十分に終わってしまう

**mode で画面遷移するページは特殊なコツが必要**

# 自動化するためにはどうしたら良いか？

- ~~Selenium~~ Playwright[^1]を併用して、日本語入力の壁を乗り越える！
- POST パラメータを工夫して、特殊な画面遷移の壁を乗り越える！
- 参考: [Driving OWASP ZAP with Selenium](https://owasp.org/www-chapter-london/assets/slides/OWASPLondon-OWASP-ZAP-Selenium-20180830-PDF.pdf)


# 自動化するにあたって検討したこと

- Selenium を書く必要があるが、どんな言語がよいか？
  - EC-CUBEでは、 [Codeception](https://codeception.com) で記述された既存のE2Eテストもあり、二重にテストを書くことになる
- OWASP ZAP API SDK の PHP 版は2016年から更新されていないため、不安が残る🤔

**npm や yarn を使用してのビルドは scss やEC-CUBE2系で実績があるので TypeScript でテストを書こう！**

- こちらの Pull Request に詳しく記載しています → https://github.com/EC-CUBE/ec-cube/pull/5169


# 自動化の方針

- OWASP ZAP の API と ~~Selenium~~ Playwright[^1] を使用してアクティブスキャンを実行する。おおまかな流れは以下の通り
  1. OWASP ZAP の API でコンテキストや、自動ログインを設定する
  2. ~~Selenium~~ Playwright[^1] で OWASP ZAP の Proxy を通してクロールする
  3. クロールしたページに対してアクティブスキャンを実行する
  4. OWASP ZAP のセッションを保存する
- 毎週1回0時に実行する
- High 以上のアラートが出た場合は、 GitHub Actions がエラーとなる
  - 誤検知の場合は alertfilter に追加するオペレーションが必要
- GitHub Actions のワークフローが完了すると、 OWASP ZAP のセッションデータがアップロードされ、GitHub Actions の Artifacts からダウンロードできる。これをローカル環境の OWASP ZAP で開くことで、アラートの内容などを確認することができる

# 自動化のデモ

- GUIモードとCUIモードがある
- 途中経過を確認したい場合は GUI モード
  - GUIモードでは OWASP ZAP のツール→オプション→API にてクライアントのIPアドレスを許可する必要がある
- GitHub Actions では CUI モード
  - 開発中にスキャンの途中で操作したい場合は API を使う。 http://zap/UI から Webインターフェイスにアクセスできる

実行方法は以下のREADMEを参照ください
https://github.com/EC-CUBE/ec-cube/tree/4.1/zap/selenium/ci/TypeScript#automated-security-tests-with-owasp-zap


# 実行する上での留意事項

- 必ず **プロテクトモード** で実行してください
- macOS だと大変遅いです
  - そもそも Symfony と Docker for Mac の相性があまりよろしくない
  - 私の環境は Windows11 + WSL2 です。16コア 32スレッドのCPU、 M.2 NVMe SSD で殴ってます

# EC-CUBE2系での侵入テスト自動化の開発について

- そもそもE2Eテストもあまり充実して無いので、、、
- いっそのこと、 E2Eテストと侵入テストを同時に実行できるようにしてしまえ！
- 開発中の Pull Request をご覧ください
  - [【ご意見ください】OWASP ZAP のアクティブスキャンとE2Eテストをまとめて実行する試み ](https://github.com/EC-CUBE/ec-cube2/pull/482)

# OWASP ZAP のお話はここまでです

*続けてEC-CUBEを安全に構築するための大切なお話です*

# EC-CUBEを安全に構築するための対策

- そもそも、どのような攻撃から守る必要があるのか？
- 攻撃から守るためには何をしたら良いか？

# 主な攻撃の手口は...？

- 見えちゃいけないファイルが見えていた
- [Water Pamola](https://blog.trendmicro.co.jp/archives/27875)
- 同居している CMS や CGI の脆弱性を利用した攻撃
- OSコマンドインジェクション
- SQLインジェクション

- *詳しくは[徳丸さんの Youtube チャンネル](https://www.youtube.com/channel/UCLNW6Bo_YU3TxnzsII2gEDA)を見てください*
- *EC-CUBE DAY 2021 に参加された方は、徳丸さんのセッションのバックナンバーを是非見てください*

# ファイル改竄を防ぐにはどうしたら良いか？

- そもそも、XSS などの脆弱性を出さないようにする
- **Webサーバーのユーザー権限で、テンプレートや JavaScript ファイルを変更させないこと**
- **Webサーバーのユーザー権限で、書き込める範囲を最小限にすること**

## そもそも、ユーザー権限って？

「誰が」、「どのような機能を動すか」を定める権限です。

例えば、 `ssh taro@example.jp` でログインした場合は、`taro` さんのユーザー権限でコマンドを実行します。
VPS やクラウドなどのWebサーバーは、 `apache` や `www-data` などのユーザー権限が割り当てられている場合が多いです。
レンタルサーバーは、FTP(S)/SSHのユーザー権限(`taro` さんのユーザー権限)でWebサーバーを動かしている場合が多いです。

# Webサーバーのユーザー権限で、不正な書き込みを防ぐためにはどうしたら良いか？

- ファイルのアップロード、変更はできる限り FTP(S)/SSH を使用することにして、とにかく Webサーバー権限で書き込める範囲を最小限にする
  - **デザイン管理画面を使っていない場合は閉鎖する**
  - **ファイル管理画面も使っていない場合は閉鎖する**
  - **プラグイン、デザインテンプレートのアップロード機能は使う時以外閉鎖する**
- 顧客にファイルアップロードを許可するカスタマイズをした場合は要注意
- ファイルの変更は FTP(S)/SSH ユーザー権限で行う
- 静的ファイルのパーミッションは 604(環境によって変わります)
- **多くの共有レンタルサーバーでは FTP(S)/SSH ユーザー権限で Webサーバーが動作しているため、対策できない**
  - VPS や IaaS のクラウド環境の方が安全性を高めやすい

# Webサーバーのユーザー権限で、書き込める範囲を最小限にするには

- 利用頻度の低い機能は閉じる
- Webサーバーの権限で書き込める箇所は、以下に限定する
  - 画像フォルダ(html/upload など)
  - キャッシュフォルダ(4系: var, 3系: app/cache, 2系: data/cache, data/Smarty/templates_c)
  - httpd.conf で `AllowOverride None` にしておくのが安全
    - .htaccess を不正にアップロードされたり、改竄されても被害を最小限にするため
- プラグインのインストール等、管理画面機能の一部は動作しなくなるが、安全性を高めるためには利便性を削った方が良い
- CMS 機能は他の優秀な CMS に任せた方が安全
  - EC-CUBEのCMS機能で頑張りすぎない方がよい

# 他に注意すべきこと

- PHPはデフォルトの設定でファイルやディレクトリを生成した時点では、すべてのユーザーが書き込める状態なので注意(ファイル:666, ディレクトリ:777 になってしまう)
- 過信は禁物。何重もの対策を
  - 管理画面のURLを変更したからヨシ! とか
- WAF やファイル変更監視でお守りするのは重要
- Gitでバージョン管理してバージョンアップしやすい構成にする
  - [EC-CUBEのバージョンアップを見据えたカスタマイズ方法2020年度版](https://qiita.com/nanasess/items/7ad3592073458adae09d)
  - プラグインや Customize ディレクトリだけのカスタマイズで頑張ろうとすると、複雑化しやすく脆弱性も潜在しやすくなる
  - 本体にはパッチを当てたけど、プラグインは当たってなかった等の事故を防ぐ
  - 管理画面からファイル更新するとバージョン管理されない...

最近は、管理画面にIP制限していても防げないんです

https://twitter.com/ockeghem/status/1412357355228995585

プラグインや Customize ディレクトリで頑張りすぎるのも禁物

https://twitter.com/nanasess/status/1463343851112779780


# そのほか

- .htaccess を改竄されるケースもある
- 画像ファイルに偽装したバックドアを設置されたケースもある
- .env も見られないように注意
- .git/config なども注意
- Nginx など .htaccess の効かない環境は注意
- [WordPress の安全性を高める記事](https://wpdocs.osdn.jp/WordPress_の安全性を高める)は、とても参考になります

# EC-CUBEを安全に構築するための対策まとめ

- 見えちゃいけないファイルは確実に隠そう
- Webサーバーの書き込み権限は最小限に
- 利便性と安全性はトレードオフ。管理画面の機能を限定することも選択肢に
- カスタマイズ時の脆弱性に注意。 Git でのバージョン管理は大切
- 過信は禁物。何重もの対策をしよう

# ご静聴ありがとうございました！

ご質問などありましたら、 Discussion にてお知らせください！

[^1]: [EC-CUBE/ec-cube#5263](https://github.com/EC-CUBE/ec-cube/pull/5263) にて、現在は Playwright の実装に変更されました
