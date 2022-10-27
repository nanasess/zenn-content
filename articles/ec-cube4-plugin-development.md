---
title: "Composerを使いこなしてEC-CUBE4系プラグインの開発効率を爆上げする"
emoji: "🐎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ECCUBE", "Plugin"]
published: true
---
EC-CUBE4系のプラグイン開発、みなさんどうされていますか？

1. `bin/console eccube:plugin:generate` コマンドでプラグイン生成
2. `bin/console eccube:plugin:install --code=<PluginCode>` でインストール
3. `app/Plugin/<PluginCode>` に移動して、 git でバージョン管理しつつ開発

という感じでしょうか？
オーナーズストアに対応する場合は、 [mock-package-api](https://doc4.ec-cube.net/plugin_mock_package_api) を使用して、一旦 tgz で固めないといけませんね。

開発中「あ、プラグイン削除のテストしなきゃ」とプラグインを削除したら、.git もろともプラグインが削除されて、泣きそうになった経験はありませんか？
(僕は何回かあります...)

このような背景から EC-CUBE4系プラグインの開発をする場合は少々工夫が必要で、日々ストレスを募らせており、何とか解消できないかと悶々としておりました。


## Composer と EC-CUBE4系プラグインの関係

EC-CUBE4系のプラグインは Composer を利用して管理されてます。

通常、 Composer は [packagist.org](https://packagist.org/) にあるリポジトリと通信して、一覧やパッケージの詳細、バージョン情報、依存関係などを取得します。
EC-CUBE4系には EC-CUBE Package API の Composer リポジトリが追加されており、この API と通信して、有料プラグインのライセンス情報や、インストール状況などを取得します。

## Composer を使用して、EC-CUBE4系のプラグインをインストールする

EC-CUBE Package API の Composer リポジトリは、通常の Composer リポジトリの機能拡張です。
機能拡張されている部分は、主に EC-CUBE の管理画面から操作できるよう拡張されている部分ですので、コマンドライン操作に限ってしまえば、通常の Composer リポジトリとほとんど変わりません。

ですので、 composer.json の `repositories` にEC-CUBE4系プラグインのリポジトリを追加してあげれば、`bin/console eccube:composer:require` コマンドを使用して、プラグインをインストール可能です。

## Composer のローカルリポジトリを活用する

Composer には、[ローカル環境のディレクトリを参照できる](https://getcomposer.org/doc/05-repositories.md#path)機能がありますので、これを活用してみます。

**この方法は、EC-CUBE4.0.x〜4.2.x まで使用可能です。**

### 事前準備

`bin/console eccube:plugin:generate` コマンドでプラグインの雛形を生成しておきます

```shell
bin/console eccube:plugin:generate


EC-CUBE Plugin Generator Interactive Wizard
===========================================

 name [EC-CUBE Sample Plugin]:
 > <enter>

 code [Sample]:
 > <enter>

 ver [1.0.0]:
 > <enter>
```

上記で `app/Plugin/Sample` というプラグインの雛形が生成されます。
これを、 EC-CUBE と同階層へ移動しておきます。

```
mv app/Plugin/Sample ../Sample
```

### composer.json にリポジトリを追加する

composer.json の `repositories` に、先ほどの Sample プラグインのローカルリポジトリを追加します。

オーナーズストアのプラグインと同じパッケージ名のプラグインを開発する場合は、Package API のリポジトリに `exclude` を追加してください。
`exclude` に記述したプラグインは、Package API のリポジトリの検索対象から除外されます。

`"exclude": ["ec-cube/sample", "ec-cube/recommend42"],` という感じで複数記述できます。

*// ここから // ここまで は追加時には削除してください。構文エラーになってしまいます*

```json
    "repositories": {
    // ここから
        "sample-plugin": {
            "type": "path",
            "url": "../Sample"
        },
    // ここまで
        "eccube": {
            // ここから
            "exclude": ["ec-cube/sample"],
            // ここまで
            "type": "composer",
            "url": "...",
            "options": {
                "http": {
                    "header": ["X-ECCUBE-KEY: ..."]
                }
            }
        }
    }
```

`repositories` の項目自体が無い場合は、EC-CUBE の管理画面→オーナーズストア→認証キー設定にて、認証キーを登録すると生成されると思います。

### ローカルリポジトリのプラグインをインストール

`bin/console eccube:composer:require` コマンドを利用してプラグインをインストールします。

```shell
bin/console eccube:composer:require ec-cube/sample
```

たくさんログが出力されますが、以下のようなログが含まれていればインストール成功です。

```
[81.2MiB/12.76s] Package operations: 1 install, 0 updates, 0 removals
[81.3MiB/12.77s]   - Installing ec-cube/sample (1.0.0): [81.3MiB/12.77s] Symlinking from ../Sample[81.3MiB/12.77s]
```

インストール時に以下のようなエラーが出た場合は、

```
In PluginInstaller.php line 56:

  Undefined index: id
```

プラグインの composer.json に `extra.id` を追加してください。(id の値は重複しない数字であればOKです)

```
diff --git a/composer.json b/composer.json
index 7e23bbd..2d1cf32 100644
--- a/composer.json
+++ b/composer.json
@@ -7,6 +7,7 @@
     "ec-cube/plugin-installer": "^2.0"
   },
   "extra": {
-    "code": "Sample"
+      "code": "Sample",
+      "id": 1
   }
 }
```

`app/Plugin` を確認すると、シンボリックリンクでプラグインが追加されていると思います。

```
ls -al app/Plugin
drwxr-sr-x 12 nanasess nanasess 4096 10月 17 16:08 .
drwxr-sr-x 10 nanasess nanasess 4096  7月  5 09:54 ..
lrwxrwxrwx  1 nanasess nanasess   26 10月 17 16:08 Sample -> ../../../Sample/
```

シンボリックリンクですので、管理画面からプラグインを削除してもリンク先のソースコードが消えることはありません。
このインストール方法は、独自プラグインにも、オーナーズストアのプラグインにも使えます。

## 注意点

- Composer API でプラグイン一覧を返すことができないので、管理画面の **プラグインを探す** からはインストールできません。(有効化/無効化/削除は可能)
- 開発用途なので、本番環境での使用は推奨しません。

## その他

EC-CUBE4.2.1 からは、[本インストール方法を活用したオプション](https://github.com/EC-CUBE/ec-cube/pull/5843) が追加されます。

また [PHPバージョンごとに Docker イメージを作成する](https://github.com/EC-CUBE/ec-cube/pull/5801)機能を活用すると、E2Eテストも10分くらいで書けてしまいます。

https://zenn.dev/nanasess/articles/ec-cube-plugin-e2etesting-in-10mins

**Composerを使いこなしてEC-CUBE4系プラグインの開発効率を爆上げして睡眠時間を増やしましょう！**
