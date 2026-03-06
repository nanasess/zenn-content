---
title: "macOS のシステムデータが578GB!? Cloudflare WARP のクラッシュレポートが原因だった"
emoji: "💾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["macOS", "Cloudflare", "APFS", "ストレージ"]
published: true
---

:::message
この記事は [Claude Code](https://docs.anthropic.com/en/docs/claude-code) と共に調査・解決した内容を元に執筆しました。
:::

macOS のシステム環境設定でストレージを確認すると、「システムデータ」が **578GB** も消費していました。1TB の SSD のうち半分以上がシステムデータとは明らかに異常です。

この記事では、原因の特定から解決までの手順を紹介します。

## 環境

- macOS Tahoe 26.3
- Cloudflare WARP クライアントをインストール済み
- APFS フォーマット、1TB SSD

## 原因の特定

### ディスク使用量の調査

まず、どのディレクトリが容量を消費しているか調べます。

```bash
du -sh /Library
# 410G /Library
```

`/Library` が **410GB** と異常に大きいことがわかりました。さらに掘り下げます。

```bash
du -d1 -h /Library/Application\ Support | sort -hr | head -5
```

```
397G /Library/Application Support
388G /Library/Application Support/Cloudflare
```

```bash
du -d1 -h /Library/Application\ Support/Cloudflare | sort -hr
```

```
388G /Library/Application Support/Cloudflare
384G /Library/Application Support/Cloudflare/crash_reports
  0B /Library/Application Support/Cloudflare/snapshots
  0B /Library/Application Support/Cloudflare/qlogs
```

**Cloudflare WARP のクラッシュレポートが 384GB** 蓄積されていました。

## 解決手順

### 1. Cloudflare WARP の削除またはクラッシュレポートの削除

Cloudflare WARP 自体が不要であればアンインストールします。クラッシュレポートだけを削除する場合は以下のコマンドを実行します。

```bash
sudo rm -rf /Library/Application\ Support/Cloudflare/crash_reports/*
```

### 2. ストレージが解放されない場合：APFS スナップショットの確認

Cloudflare WARP を削除（またはクラッシュレポートを削除）しても、macOS のストレージ表示が変わらないことがあります。これは **APFS スナップショット**が削除前の状態を保持しているためです。

#### Time Machine ローカルスナップショットの削除

```bash
# スナップショットの一覧
tmutil listlocalsnapshots /

# 全スナップショットを一括削除
sudo tmutil deletelocalsnapshots /
```

#### ASR スナップショットの確認

Time Machine のスナップショットを削除しても解放されない場合、**Apple Software Restore（ASR）スナップショット**が残っている可能性があります。

```bash
diskutil apfs listSnapshots disk1s2
```

出力例:

```
Snapshot Name: com.apple.asr.254
Snapshot UUID: CB2895FA-DE9E-467F-912C-208C88FF13C8
```

`com.apple.asr.*` という名前のスナップショットは、macOS アップデート時に作成されるロールバック用のスナップショットです。このスナップショットが Cloudflare の 384GB を含んだ状態で保持されているため、ファイルを削除しても空き容量が戻りません。

```bash
sudo diskutil apfs deleteSnapshot disk1s2 -name com.apple.asr.254
```

:::message alert
ASR スナップショットを削除すると、直前の macOS アップデートへのロールバックができなくなります。現在の macOS が正常に動作していることを確認してから削除してください。
:::

### 3. 解放の確認

```bash
diskutil info disk1s2 | grep "Volume Used Space"
```

削除前後で使用量が大幅に減少していれば成功です。

## なぜストレージ表示と実際の空き容量がずれるのか

macOS の APFS には以下の仕組みがあり、ファイルを削除しても即座にストレージが解放されない場合があります。

| 仕組み | 説明 |
|--------|------|
| **Time Machine ローカルスナップショット** | Time Machine が定期的に作成するスナップショット。削除前のデータを保持する |
| **ASR スナップショット** | macOS アップデート時に作成されるロールバック用スナップショット |
| **パージ可能領域** | APFS が削除済みデータを即座に解放せず、パージ可能としてマークする場合がある |

`du` コマンドで見える合計と `diskutil info` の使用量に大きな差がある場合、これらのスナップショットが原因であることが多いです。

## まとめ

- macOS の「システムデータ」が異常に大きい場合、まず `/Library` 配下を `du` で確認する
- Cloudflare WARP はクラッシュレポートを大量に蓄積することがある
- ファイルを削除してもストレージが解放されない場合は、APFS スナップショット（Time Machine + ASR）を確認する
- 特に `com.apple.asr.*` スナップショットは `tmutil` では削除できないため、`diskutil apfs deleteSnapshot` を使う
