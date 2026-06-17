---
title: "Monolog 1.x が php-fpm 下で古いログを消さない問題（2.x/3.x との挙動差）"
emoji: "🗑️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["monolog", "php", "symfony", "eccube", "logging"]
published: true
---

:::message
この記事は Claude Code で執筆しています。
:::

## TL;DR

- Monolog の `RotatingFileHandler` には `max_files` で古いログを自動削除する仕組みがある。
- ところが **Monolog 1.x** は、この削除（GC）を `close()` / `__destruct()` のタイミングでしか行わない。
- そのため **php-fpm / mod_php のような「1 リクエスト 1 プロセス」の Web 実行環境**、特に `fingers_crossed` でラップした構成では、削除が確実に発火せず**古いログが無限に溜まり続ける**ことがある。
- **Monolog 2.x / 3.x** は `write()` の直後に同期的にローテートするよう修正されており、この問題は起きない。
- バージョンを上げられない場合は、Monolog の `max_files` に頼らず **logrotate か cron の `find -mtime +N -delete`** で削除するのが確実（Monolog 公式も docblock で logrotate を推奨している）。

## 起きたこと

EC-CUBE 4.1 系の本番環境（Apache + php-fpm）で、`var/log/prod/` 配下の日次ログが**まったく削除されず**ディスクを圧迫していました。

設定上は `max_files: 60` が入っており、60 世代を超えたら古いものから消える**はず**です。

```yaml
# app/config/eccube/packages/prod/monolog.yml（抜粋）
monolog:
    handlers:
        main:
            type: fingers_crossed
            action_level: error
            handler: main_rotating_file
        main_rotating_file:
            type: rotating_file
            max_files: 60
            path: '%kernel.logs_dir%/%kernel.environment%/site.log'
            level: warning
```

しかし実際には、半年分以上のファイルが 1 日 1 個ずつ、1 件も消えずに残っていました。

```console
$ ls -1 var/log/prod/site-*.log | wc -l
167
$ ls -la var/log/prod/site-2026-01-01.log
-rw-r--r-- 1 apache apache 5827660  1月  1 23:53 var/log/prod/site-2026-01-01.log
```

`167` という数字は、最初のファイルの日付から起算した**日数とぴったり一致**します。つまり **GC は一度も削除に成功していない**ということです。

## まず外れた仮説たち

最初に疑ったのは「権限」と「glob 失敗」でしたが、いずれも**外れ**でした。記録として残しておきます。

`RotatingFileHandler::rotate()`（削除処理）はおおむね次の構造です（Monolog 1.x）。

```php
protected function rotate()
{
    $this->url = $this->getTimedFilename();
    $this->nextRotation = new \DateTime('tomorrow');

    if (0 === $this->maxFiles) {
        return;                       // ← max_files: 0 なら何もしない
    }

    $logFiles = glob($this->getGlobPattern());
    if ($this->maxFiles >= count($logFiles)) {
        return;                       // ← 60 >= 167 は false なので通過
    }

    usort($logFiles, function ($a, $b) {
        return strcmp($b, $a);        // 名前降順（新しい順）
    });

    foreach (array_slice($logFiles, $this->maxFiles) as $file) {
        if (is_writable($file)) {     // ← 書き込み可否でガード
            set_error_handler(function () {});
            unlink($file);
            restore_error_handler();
        }
    }
    $this->mustRotate = false;
}
```

ここから「サイレントに削除をスキップする」分岐は次の 3 つです。

1. `max_files: 0`（無制限）→ 今回は `60` なので該当しない。
2. `glob()` が `false` → `disable_functions` の `glob` や `open_basedir` 制限が原因になりうる。
3. `is_writable($file)` が `false` → 別ユーザー所有のファイルは削除されない。

本番で確認したところ:

```console
$ php -i | grep -Ei 'disable_functions|open_basedir'
disable_functions => no value => no value
open_basedir => no value => no value

$ php -r "var_dump(glob('var/log/prod/site-*.log') ?: 'GLOB_FAILED');"
array(167) { ... }                    # glob は正常に 167 件返す
```

ファイル所有者もすべて `apache apache` の `0644` で統一されており、GC を走らせる apache 自身が所有者なので `is_writable()` は `true`。

つまり **`rotate()` さえ走れば確実に削除されるはず**なのに、それが起きていない。問題は「削除処理の中」ではなく「**`rotate()` がそもそも呼ばれていない**」ことにありました。

## 削除が走る条件

`rotate()` が呼ばれる経路は、`close()` の中だけです（Monolog 1.x）。

```php
public function close()
{
    parent::close();
    if (true === $this->mustRotate) {
        $this->rotate();
    }
}

protected function write(array $record)
{
    // 当日のファイルが存在しなければ「ローテートすべき」とマークする
    if (null === $this->mustRotate) {
        $this->mustRotate = !file_exists($this->url);
    }
    if ($this->nextRotation < $record['datetime']) {
        $this->mustRotate = true;
        $this->close();
    }
    parent::write($record);
    //  ↑ ここで終わり。write() の後に close() を呼ぶ行が無い！
}
```

ポイントは、**1.x の `write()` は書き込み後に `close()` を呼ばない**ことです。`mustRotate` を `true` にセットするだけで、実際の削除は「いつか `close()` が呼ばれたとき」に先送りされます。

では `close()` はいつ呼ばれるのか？ Monolog の `AbstractHandler` はデストラクタで `close()` を呼びます。

```php
public function __destruct()
{
    try {
        $this->close();
    } catch (\Throwable $e) {
        // do nothing
    }
}
```

つまり 1.x の GC は、最終的に**オブジェクト破棄（`__destruct`）任せ**です。

### fingers_crossed がさらに状況を悪くする

今回は `rotating_file` を `fingers_crossed` でラップしています。`FingersCrossedHandler::close()` を見ると:

```php
public function close()
{
    $this->flushBuffer();   // バッファを内側ハンドラへ流すだけ
}
```

`flushBuffer()` は内側ハンドラの `handleBatch()`（＝`write()`）を呼ぶだけで、**内側ハンドラの `close()` は呼びません**。

結果として、`RotatingFileHandler` の `rotate()` を起動できるのは「自分自身の `__destruct`」だけになります。リクエスト終了時のシャットダウン処理でこの `__destruct` が `mustRotate = true` の状態で確実に発火しないと、削除は永遠に行われません。

## 再現実験

「本当にバージョンで挙動が変わるのか？」を、推測せず実際に確かめました。

- 166 個の古い日付ファイルをあらかじめ用意（本番の積み上がりを再現）
- `fingers_crossed(action_level: error)` → `rotating_file(max_files: 60, level: warning)` を構成
- `info` / `warning` / `error` を 1 回ずつログ出力
- 各タイミングでファイル数をカウント

```php
<?php
require __DIR__.'/vendor/autoload.php';

use Monolog\Logger;
use Monolog\Handler\RotatingFileHandler;
use Monolog\Handler\FingersCrossedHandler;

$dir = sys_get_temp_dir().'/montest_'.uniqid();
mkdir($dir);
$base = $dir.'/site.log';

// 166 個の古い日付ファイルを用意
$d = new DateTime('2025-01-01');
for ($i = 0; $i < 166; $i++) {
    file_put_contents($dir.'/site-'.$d->format('Y-m-d').'.log', "old\n");
    $d->modify('+1 day');
}
echo "事前ファイル数: ", count(glob($dir.'/site-*.log')), "\n";

$rot = new RotatingFileHandler($base, 60, Logger::WARNING);
$fc  = new FingersCrossedHandler($rot, Logger::ERROR);
$logger = new Logger('test');
$logger->pushHandler($fc);

$logger->info('buffered info');
$logger->warning('buffered warn');
$logger->error('boom');                 // ここで fingers_crossed が発火
echo "error 直後: ", count(glob($dir.'/site-*.log')), "\n";

$fc->close();
echo "fc->close() 後: ", count(glob($dir.'/site-*.log')), "\n";

unset($logger, $fc, $rot);
gc_collect_cycles();                     // __destruct を強制発火
echo "__destruct 後: ", count(glob($dir.'/site-*.log')), "\n";
```

`monolog/monolog` のバージョンだけを差し替えて実行した結果がこちらです。

| タイミング | **1.26.1** | **2.11.0** | **3.10.0** |
|---|---|---|---|
| `error` ログ直後 | **167** | **60** | **60** |
| `fc->close()` 後 | **167** | 60 | 60 |
| `__destruct` 後 | **60** | 60 | 60 |

読み取れること:

- **1.26.1**: `write()` でも `fingers_crossed::close()` でも消えず、**`__destruct` まで来てようやく 60 に削除**される。
- **2.11.0 / 3.10.0**: `error` を吐いた**その瞬間に 166 → 60** へ同期削除される。

## なぜバージョンで変わるのか

2.x で `write()` に**書き込み直後のローテート**が追加されたためです。

```php
// Monolog 2.11.0 RotatingFileHandler::write()
protected function write(array $record): void
{
    if (null === $this->mustRotate) {
        $this->mustRotate = null === $this->url || !file_exists($this->url);
    }
    if ($this->nextRotation <= $record['datetime']) {
        $this->mustRotate = true;
        $this->close();
    }
    parent::write($record);
    if ($this->mustRotate) {
        $this->close();        // ← 2.x で追加。書き込み直後に同期ローテート
    }
}
```

3.x も同様に `parent::write()` の直後で `close()` を呼びます（型が `LogRecord` になっただけ）。

この 1 行があることで、削除は「`__destruct` のタイミング」に依存しなくなり、**ログを書いた瞬間に確実に走る**ようになります。逆に 1.x はこの行が無いため、削除がオブジェクト破棄任せになり、後述のとおり php-fpm 環境では発火しないことがあります。

## なぜ php-fpm で表面化するのか

再現実験の 1.26.1 は、最後に `unset()` + `gc_collect_cycles()` で**破棄を明示的に強制**したため、最終的に削除されました。CLI でワンショット実行するぶんには 1.x でも消えるわけです。

しかし php-fpm / mod_php のような Web 実行環境は事情が違います。

- ハンドラは Symfony の DI コンテナが保持しており、リクエストの間ずっと参照が残る。
- `fingers_crossed` 構成だと、内側の `rotating_file` の `write()` が呼ばれるのは**バッファが flush される（＝エラー発生 or passthru）リクエスト終了間際**。
- その後の削除は内側ハンドラの `__destruct` 任せだが、リクエスト終了時のシャットダウン処理におけるデストラクタの発火順序や、`mustRotate` がセットされる前にハンドラが破棄される可能性などにより、**`rotate()` が確実には走らない**。

結果として、CLI では消えるのに **php-fpm の本番では一度も消えない**、という今回の症状になります。「1 リクエスト 1 プロセス」かつ「破棄タイミング依存」という組み合わせが、1.x の弱点を踏み抜く形です。

:::message
ここでの主役は「Web SAPI（php-fpm / mod_php）における破棄タイミング依存」です。1.x が原理的にローテート不能なのではなく、**削除を `__destruct` に先送りする設計が、Web のリクエストライフサイクルでは信頼できない**、というのが正確な理解です。2.x / 3.x はこの先送りをやめたので問題が消えています。
:::

## 対策

### 1. バージョンに依存しない確実な方法（推奨）

Monolog の `RotatingFileHandler` のクラスコメントには、はっきりこう書かれています。

> This rotation is only intended to be used as a workaround. Using logrotate to handle the rotation is strongly encouraged when you can use it.

つまり公式自身が「これは workaround、可能なら logrotate を使え」と言っています。`max_files` に頼らず、OS 側で削除するのが堅実です。

```bash
# 即時：最新 60 個を残して掃除
ls -1t var/log/prod/site-*.log | tail -n +61 | xargs -r rm -f

# 恒久：cron（Web 実行ユーザーの crontab）で 60 日分（当日含む）を残して古いログを削除
5 0 * * * find /path/to/var/log/prod -name 'site-*.log' -mtime +59 -delete
```

`find -mtime` は Monolog のバージョン・`__destruct`・`glob`・`fingers_crossed` のいずれにも依存しないため、確実に効きます。

### 2. Monolog を 2.x / 3.x に上げる

`write()` 直後の同期ローテートが入っているので、この問題自体は解消します。ただしフレームワークやプラグインが 1.x の API（処理シグネチャ等）に依存している場合、メジャーアップは単純な差し替えにはなりません。`symfony/monolog-bridge` / `symfony/monolog-bundle` の対応バージョンや、自前ハンドラ・プロセッサの互換性を確認したうえで、テストを伴う移行として扱う必要があります。

## まとめ

- `max_files` が効かない原因は、権限でも `glob` でもなく **Monolog のバージョン差**だった。
- **Monolog 1.x** は古いログの削除を `close()` / `__destruct` 任せにしており、**php-fpm + fingers_crossed** ではその発火が信頼できず、ログが無限に溜まる。
- **Monolog 2.x / 3.x** は `write()` 直後に同期ローテートするため、この問題は起きない（再現実験で確認）。
- バージョンを上げられないなら、**logrotate か cron の `find -mtime +N -delete`** で削除するのが確実。Monolog 公式の推奨もこれ。

ログローテーションを「設定したから安心」と思い込まず、本番で**実際に古いファイルが消えているか**を一度確認しておくことをおすすめします。
