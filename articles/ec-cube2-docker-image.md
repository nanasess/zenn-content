---
title: "EC-CUBE2系の Docker イメージを作りました"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "ECCUBE", "ECCUBE2", "Docker"]
published: true
---
EC-CUBE4系がリリースされていますが、EC-CUBE2系を運用されている方も多いと思います。
2.17系が最新ですが、決済やプラグインの開発者の方は、2.11系や2.13系などの古い環境で開発しなくてはならない場合も多々あります。

最近は macOS で古いPHPがビルドできなくなったり、CentOS が EOL を迎えると、PHP5系の環境を用意するのも大変苦労します。。。

これを改善すべく、EC-CUBE2系の Docker イメージを作ってみました。
docker-compose コマンド一発で、モジュールを即開発できます。

**10分でEC-CUBEプラグインのE2Eテストを書いてみる記事**を活用すれば、EC-CUBE2.11系などの古い環境でもE2Eテストを簡単に書けてしまいます。

https://zenn.dev/nanasess/articles/ec-cube-plugin-e2etesting-in-10mins

# 使用方法

以下のリポジトリにまとめましたので、ぜひ git clone してお試しください。

https://github.com/nanasess/ec-cube2_mdl_sample

## 対応バージョン

docker-compose.yml の `image:` を使用したいバージョンに合わせて変更してください。

```yaml
## 例) EC-CUBE 2.12.6 をPHP 5.5 で使用したい場合
image: ghcr.io/nanasess/ec-cube2-php:5.5-apache-2.12.6
```

### PHPバージョン

- 5.4
- 5.5
- 5.6

### EC-CUBEバージョン

- 2.11.5
- 2.12.6
- 2.13.5
- 2.17.x

2.17.x は、公式の Docker イメージ(PHP7.4-8.4)があります

```yaml
## 例) EC-CUBE 2.17.x をPHP 8.0 で使用したい場合
image: ghcr.io/nanasess/ec-cube2-php:8.0-apache
```

## Usage

Docker コンテナが起動したら、 https://localhost:4430 にアクセスしてください。
管理画面は https://localhost:4430/admin にアクセスしてください。
(ID:admin, PASS: password)

### PostgreSQL を使用する場合

```
docker compose -f docker-compose.yml -f docker-compose.pgsql.yml -f docker-compose.dev.yml up -d
```

### MySQL を使用する場合

```
docker compose -f docker-compose.yml -f docker-compose.mysql.yml -f docker-compose.dev.yml up -d
```

## サンプルモジュールについて

このサンプルは、EC-CUBE2系モジュールのサンプルを含んでいます。
サンプルモジュールがインストール済みの状態で、Docker コンテナが起動します。

**管理画面→オーナーズストア→購入商品一覧→購入商品一覧を取得する** をクリックし、一覧の **設定** をクリックすると、サンプルモジュールの設定画面にアクセスできます。

このリポジトリは、 EC-CUBEの `data/downloads/module/mdl_sample` にマウントされています。
[config.php](./config.php) を修正することで、モジュールの設定画面を編集可能です。

## Docker コンテナ起動時の設定

Docker コンテナ起動時に任意のSQLを実行したい場合は、 [dockerbuild/sql/setup.sql](./dockerbuild/sql/setup.sql) を編集してください。

また [docker-compose.dev.yml](./docker-compose.dev.yml) の `entrypoint` を修正することで、起動時にスクリプトを実行できます。

## 注意事項

macOS で 2.17.x のイメージを起動しようとすると、以下のようなエラーと共に ec-cube が終了してしまう場合があります。

``` shell
2025-01-14 17:12:40 chmod: changing permissions of './data/downloads/module/mdl_sample/.git/objects/pack/pack-6b6337a08a08029be8ca5dae134eb542dd6aeba1.rev': Permission denied
2025-01-14 17:12:40 chmod: changing permissions of './data/downloads/module/mdl_sample/.git/objects/pack/pack-6b6337a08a08029be8ca5dae134eb542dd6aeba1.pack': Permission denied
2025-01-14 17:12:40 chmod: changing permissions of './data/downloads/module/mdl_sample/.git/objects/pack/pack-6b6337a08a08029be8ca5dae134eb542dd6aeba1.idx': Permission denied
```

どうも、マウントしたディレクトリの中の .git のパーミッションを変更しようとした場合にエラーになるようです。

このようなエラーに遭遇した場合は、以下の PR を適用してみてください。

https://github.com/nanasess/ec-cube2_mdl_sample/pull/2

## See Also

- [Composerを使いこなしてEC-CUBE4系プラグインの開発効率を爆上げする](https://zenn.dev/nanasess/articles/ec-cube4-plugin-development)
- [10分でEC-CUBEプラグインのE2Eテストを書いてみる](https://zenn.dev/nanasess/articles/ec-cube-plugin-e2etesting-in-10mins)
