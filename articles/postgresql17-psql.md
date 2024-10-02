---
title: "PostgreSQL16以下の psql から PostgreSQL17 への接続に失敗する場合の対応方法"
emoji: "🐘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["PostgreSQL", "Database"]
published: true
---
PostgreSQL17 サーバーへ PostgreSQL17未満の psql で `\l` を実行すると以下のエラーが発生します。

```shell
psql --version
psql (PostgreSQL) 16.4

psql -h 127.0.0.1 -p 5432 -U postgres -d template1 -c 'SELECT version()'
                                                       version
---------------------------------------------------------------------------------------------------------------------
 PostgreSQL 17.0 (Debian 17.0-1.pgdg120+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14) 12.2.0, 64-bit

psql -h 127.0.0.1 -p 5432 -U postgres -d template1 -c '\l'
ERROR:  column d.daticulocale does not exist
行 8:   d.daticulocale as "ICU Locale",
        ^
HINT:  Perhaps you meant to reference the column "d.datlocale".
```
*`psql -l`オプションでも同様です。*

## エラーの原因

PostgreSQL17 から `pg_database.daticulocale` が `pg_database.datlocale` に変更されたのが原因です。

参考) https://www.postgresql.org/docs/17/release-17.html#RELEASE-17-MIGRATION

## 対応方法

PostgreSQL17 の psql を使いましょう

[docker compose の公式ドキュメント](https://docs.docker.jp/v17.06/compose/startup-order.html) に倣って、 psql コマンドで死活監視している場合は、 `SELECT 1` などを使用すると良いです。

```shell
#!/bin/bash
# wait-for-postgres.sh

set -e

host="$1"
shift
cmd="$@"

until psql -h "$host" -U "postgres" -c 'SELECT 1'; do
  >&2 echo "Postgres is unavailable - sleeping"
  sleep 1
done

>&2 echo "Postgres is up - executing command"
exec $cmd
```
