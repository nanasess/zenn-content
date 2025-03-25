---
title: "EC-CUBE2系から2.25へのバージョンアップ方法"
emoji: "🐉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["eccube"]
published: true
---

*[2.17へバージョンアップしたい方はこちらをご覧ください](https://qiita.com/nanasess/items/ff9bbde34f7d44240c06)*

EC-CUBE2.25 は、 EC-CUBE2.13.5 を PHP7 及び PHP8 に対応したバージョンです。
EC-CUBE2.13.5 に比べ、 [MySQL で約200%の高速化等、性能面でも大幅な改善](https://github.com/EC-CUBE/ec-cube2/pull/296) されており、大量のトラフィックを支える大規模サイトや、共有レンタルサーバーなどで運用したい方におすすめです。

また、フレームワークによるサポート期限の縛りがありませんので、 **EC-CUBEでは一番長くサポートされるバージョン** となっています。3年以上保守したいサイトにも適しています。
(RHEL9 と組み合わせれば、 PHP8.0 は2032年5月までサポートされます。また2025年半ばにリリースが予定されている RHEL10 と組み合わせれば、2035年までサポートされます)

2.13系や、2.17系からデータベースの変更はありません。
2.4系から移行する場合は、[データ移行プラグイン](https://www.ec-cube.net/products/detail.php?product_id=1017)をお使いください。

- **必ずローカル環境またはテスト環境で実施してください**
- 2.13.5 や2.17.2 からデータベースの変更はありません。
  - [2.11.x 及び 2.12.x からバージョンアップする際のデータベース変更は、 Seasoft さんのサイトが詳しいです](http://seasoft.jp1.cx/ec/%E3%82%A2%E3%83%83%E3%83%97%E3%83%87%E3%83%BC%E3%83%88/2.12/2.11_2.12)
- Git を使用してバージョン管理していないサイト向けの方法です。
- 既に Git を導入されている方は [著作権表示等を置換](#著作権表示等を置換) から進めると良いでしょう。
- [2.13.5 と 2.25 の差分はこちら](https://github.com/EC-CUBE/ec-cube2/compare/eccube-2.13.5...master)

[著作権表記変更](https://github.com/EC-CUBE/ec-cube2/pull/239) の影響もあり、2.13.5 からの差分はほぼ全ファイルに及びます。
また、 Composer でパッケージ管理をするようになったため、単純に差分ファイルを上書きしただけでは動作しませんのでご注意ください。

## 導入バージョンの EC-CUBE をクローン

```shell
# EC-CUBE 2.13.2 の例です
git clone https://github.com/EC-CUBE/ec-cube2.git
git fetch origin --tags
git checkout -b 2.13.2 refs/tags/eccube-2.13.2
```


EC-CUBE はリリースごとにタグがついています。
`refs/tags/<導入バージョン>` で checkout するのがポイント

## カスタマイズ済みの EC-CUBE を上書き

サーバー上のものを FTP や SCP などでダウンロードするなりして上書きしてください。
html の中に data を入れているサイトもあるかと思いますが、 先に clone した EC-CUBE と data ディレクトリの階層が一致していれば大丈夫です。

ssh が利用可能な場合は、以下のようにして上書きコピーできます。

```shell
cd ec-cube2
scp -rp user@hostname:/path/to/ec-cube/* .
```


## 著作権表示等を置換

**LOCKON CO.,LTD.** が **EC-CUBE CO.,LTD.** になっていたり、バージョン番号がついていたりしますので、コンフリクトしないように置換しておきます。
普段お使いのエディタを使用しても大丈夫です。

```shell
export LANG=C # Mac の BSD sed を使う場合のおまじない

# 著作権表示を置換
find . -name '*.php' -print0 -or -name '*.tpl' -print0 -or -name '*.css' -print0 -or -name '*.js' -print0 | xargs -0 sed -i.bak 's/Copyright(c) .* LOCKON CO.,LTD. All Rights Reserved./Copyright(c) EC-CUBE CO.,LTD. All Rights Reserved./'

find . -name '*.php' -print0 -or -name '*.tpl' -print0 -or -name '*.css' -print0 -or -name '*.js' -print0 | xargs -0 sed -i.bak 's|http://www.lockon.co.jp/|http://www.ec-cube.co.jp/|'

find . -name '*.php' -print0 -or -name '*.tpl' -print0 -or -name '*.css' -print0 -or -name '*.js' -print0 | xargs -0 sed -i.bak 's|LOCKON CO.,LTD.|EC-CUBE CO.,LTD.|'

# @version タグを置換
find . -name '*.php' -print0 -or -name '*.tpl' -print0 -or -name '*.css' -print0 -or -name '*.js' -print0 | xargs -0 sed -i.bak 's|@version \$Id:.*\$|@version $Id:$|'
find . -name '*.php' -print0 -or -name '*.tpl' -print0 -or -name '*.css' -print0 -or -name '*.js' -print0 | xargs -0 sed -i.bak 's|@version \$Id:\$|@version $Id$|'

# 参照代入の修正を置換
find . -name '*.php' -print0 | xargs -0 sed -i.bak 's|$objQuery =& SC_Query_Ex::getSingletonInstance();|$objQuery = SC_Query_Ex::getSingletonInstance();|'

## tpl の置換
find . -name '*.tpl' -print0 | xargs -0 sed -i.bak 's|\(.*\)=`\(.*\)`|\1="`\2`"|g'
find . -name '*.tpl' -print0 | xargs -0 sed -i.bak 's/if $\(.*\)|@count > 0/if !empty($\1)/g'

## 改行コードを置換(管理画面から編集すると CRLF になってしまうため)
find . -name '*.php' -print0 -or -name '*.tpl' -print0 | xargs -0 sed -i.bak 's/'$'\r//'

## 最後にバックアップファイルを削除
find . -name '*.bak' -delete
```

## php-cs-fixer の適用

EC-CUBE2.25 では php-cs-fixer で PHPコードを整形しています。
旧バージョンの EC-CUBE に対しても適用できますので、これを利用することでコンフリクトを最小限にできます。

```shell
# https://github.com/PHP-CS-Fixer/PHP-CS-Fixer/releases から最新の php-cs-fixer.phar をダウンロード
curl -O -L https://github.com/PHP-CS-Fixer/PHP-CS-Fixer/releases/download/v3.64.0/php-cs-fixer.phar
# 2.25 の設定ファイルをダウンロード
curl -O -L https://raw.githubusercontent.com/EC-CUBE/ec-cube2/refs/heads/master/.php-cs-fixer.dist.php
# php-cs-fixer を適用(PHP8.4など、できるだけ新しいPHPで実行すると良いです)
PHP_CS_FIXER_IGNORE_ENV=1 php-cs-fixer.phar fix --allow-risky=yes
```

## カスタマイズ内容をコミット

ここまでの内容をコミットしておきます。
コミットしておけば、いつでも戻すことができます。

```shell
git add .
git commit -m '独自カスタマイズ'
```


## 2.25 のブランチをマージ

2.25 の変更をマージします。
独自カスタマイズした箇所以外は、自動的に変更を取り込んでくれるはず。。

```shell
git fetch origin
git merge origin/master
```

## コンフリクト解消

コンフリクトした箇所を手作業でマージします。
すべて解消したら、変更をコミットします。

**data/config/config.php など、機密情報の含まれたファイルはコミットしないでください。不用意に Github などへ push してしまうと、不正アクセスの原因となります。**

コンフリクトの解消方法は、こちらを参考にしてください → [Git を使って EC-CUBE を簡単アップデート#番外-競合コンフリクトしてしまった時は](https://qiita.com/nanasess/items/fe2a93ff64833d87eb19#番外-競合コンフリクトしてしまった時は)

## composer install コマンドを実行

EC-CUBE2.17 から、 PEAR や Smarty の外部ライブラリは、 3系、4系と同じく composer で管理するようになりました。
`git merge` が上手くいった場合は、 `data/module` の下から、これらのソースが削除されていますので、生成する必要があります。

https://getcomposer.org/download/


```shell
# composer のセットアップ
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php composer-setup.php
php -r "unlink('composer-setup.php');"
```

```shell
# composer install コマンドの実行
php composer.phar install
```

## デザインテンプレートや個別カスタマイズ箇所の修正

標準以外のデザインテンプレートを使用している場合は、 テンプレート名が default や、 sphone 以外になっていると思います。
これらは自動的にバージョンアップされませんので、 default や sphone との差分を確認しながら取り込む必要があります。

個別にカスタマイズした箇所で、 PHP8 に対応していない箇所がある場合は、個別に修正が必要です。

## プラグインの修正

2.13 に対応しているプラグインでしたら、PHP7 で非推奨になった箇所の修正さえすれば、多くは動作します。
[PHP7 で廃止された関数や構文](https://www.php.net/manual/ja/migration70.incompatible.php)や[PHP8 で廃止された関数や構文](https://www.php.net/manual/ja/migration80.incompatible.php)を使用している場合は、個別に修正する必要があります。
特に、PHP7 では Warning だった箇所が PHP8 では Fatal(システムエラー)になっていますのでご注意ください。

## data/config/config.php の修正

PHP7 になって mysql extension が廃止されたため、MySQL の場合のみ **mysqli** ドライバを使用する必要があります。
また、 `DB_PORT` は空文字を許さなくなったため、明示的に指定する必要があります。
*デフォルト値の `3306` の場合は `false` でも可*

```data/config/config.php
// mysql を mysqli に変更
define('DB_TYPE', 'mysqli');
// DB_PORT を明示的に指定
define('DB_PORT', '3306');
```

## eccube.legacy.js の追加

2.12以前からのデザインテンプレートを使用していたり、2.12系互換のプラグインなどでは、`fnModeSubmit()` など古い JavaScript 関数が使われている場合があります。
これらの互換性を向上するには、 `eccube.legacy.js`, `eccube.admin.legacy.js` といった互換用の JavaScript を追加しましょう

### フロント画面用設定例

```data/Smarty/templates/default/site_frame.tpl
<script type="text/javascript" src="<!--{$smarty.const.ROOT_URLPATH}-->js/eccube.js"></script>
<!-- ↓後方互換用 JavaScript 関数を追加 -->
<script type="text/javascript" src="<!--{$smarty.const.ROOT_URLPATH}-->js/eccube.legacy.js"></script>
```

eccube.legacy.js は以下のテンプレートに追加が必要です

- data/Smarty/templates/<テンプレート名>/site_frame.tpl
- data/Smarty/templates/<テンプレート名>/popup_header.tpl
- data/Smarty/templates/sphone/site_frame.tpl
- data/Smarty/templates/sphone/popup_header.tpl

### 管理画面用設定例

```data/Smarty/templates/admin/main_frame.tpl
<script type="text/javascript" src="<!--{$smarty.const.ROOT_URLPATH}-->js/eccube.js"></script>
<!-- ↓後方互換用 JavaScript 関数を追加 -->
<script type="text/javascript" src="<!--{$smarty.const.ROOT_URLPATH}-->js/eccube.legacy.js"></script>

<script type="text/javascript" src="<!--{$TPL_URLPATH}-->js/eccube.admin.js"></script>
<!-- ↓後方互換用 JavaScript 関数(管理画面用)を追加 -->
<script type="text/javascript" src="<!--{$TPL_URLPATH}-->js/eccube.admin.legacy.js"></script>
```

eccube.legacy.js と eccube.admin.legacy.js は以下のテンプレートに追加が必要です

- data/Smarty/templates/admin/main_frame.tpl
- data/Smarty/templates/admin/popup_header.tpl

## 動作確認

ここまでできれば、 Webサーバー上での動作確認ができると思います。
システムエラーが発生した場合は、 PHP8 に対応していないコードが含まれていると思われます。 `data/logs/error.log` をご確認ください

## その他注意事項

### 運用で使用しないファイルの削除

git merge で最新バージョンをマージすると、運用では使用しないファイルも取り込まれます。
セキュリティ向上のため、これらは削除するようにしましょう。

```shell
rm -rf ./.gitignore
rm -rf ./.github
rm -rf ./.editorconfig
rm -rf ./.php_cs.dist
rm -rf ./phpunit.xml.dist
rm -rf ./phpstan.neon.dist
rm -rf ./build.xml
rm -rf ./README.md
rm -rf ./php.ini
rm -rf ./phpinicopy.sh
rm -rf ./phpinidel.sh
rm -rf ./*.phar
rm -rf ./setup.sh
rm -rf ./svn_propset.sh
rm -rf ./playwright*
rm -rf ./e2e-tests
rm -rf ./tests
rm -rf ./templates
rm -rf ./patches
rm -rf ./docs
rm -rf ./html/test
rm -rf ./dockerbuild
rm -rf ./Dockerfile
rm -rf ./docker-compose*.yml
rm -rf ./zap
find . -name "dummy" -print0 | xargs -0 rm -rf
```

### 古いバージョンのファイルの残存に注意

data/module 以下に、2.17未満のバージョンのファイルが残存してしまう場合があります。必要なファイルは以下URLを参考にしていただき、古いファイルは削除してください

https://github.com/EC-CUBE/ec-cube2/tree/master/data/module

### html フォルダの中に data フォルダを入れている場合

ロリポップなどのレンタルサーバーで、 html フォルダの中に data フォルダを入れている場合、 2.25 の html フォルダが残存してしまう場合があります。

```
例) 2.13.5 で html フォルダを data フォルダに入れている場合
  URL: https://example.com/
  DocumentRoot(htmlフォルダ): /home/username/www/example.com
  data フォルダ: /home/username/www/example.com/data

2.25 のパッケージを、そのまま上書きすると以下のようになってしまいます。
  URL: https://example.com/
  DocumentRoot(htmlフォルダ): /home/username/www/example.com
  data フォルダ: /home/username/www/example.com/data
  2.25 の htmlフォルダ: /home/username/www/example.com/html
```

このままですと、 https://example.com/html でもアクセス可能な状態になり、セキュリティホールの原因となります。
また、 /home/username/www/example.com/define.php が 2.13.5 のままですのでシステムエラーになってしまいます。

html フォルダの中に data フォルダを入れている場合は、DocumentRoot に html フォルダが残存しないようくれぐれもご注意ください。

## どうしても動かない場合は

[開発コミュニティ](https://xoops.ec-cube.net) や、こちらのコメント覧でご相談を！
