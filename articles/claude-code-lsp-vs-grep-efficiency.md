---
title: "Claude Code で LSP(phpactor) と Grep 検索の効率を比較してみた"
emoji: "🔍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["claudecode", "LSP", "phpactor", "PHP"]
published: true
---

:::message
この記事は Claude Code で執筆しています。
:::

## はじめに

Claude Code でコードベースを調査する際、LSP（Language Server Protocol）を使った方が正確で効率的なのでは？と思い、実際に比較してみました。

対象は EC-CUBE（Symfony ベースの EC プラットフォーム）の `ItemHolderInterface` を実装しているクラスの調査です。

## 調査対象: ItemHolderInterface

`ItemHolderInterface` は、カートや注文などの「商品を保持するコンテナ」の契約を定義するインターフェースです。`PurchaseFlow` パイプラインで Cart と Order を多態的に扱うための抽象化として使われています。

主なメソッド:
- `getItems()` - ItemCollection を返す
- `getTotal()` / `setTotal()` - 合計金額
- `setDeliveryFeeTotal()` / `getDeliveryFeeTotal()` - 送料
- `setDiscount()` / `setCharge()` / `setTax()` - 値引き・手数料・税
- `addItem()` - 明細追加
- `getCustomer()` / `getShippings()` - 顧客・配送情報

## Grep 検索（LSP なし）の場合

Claude Code の Explore エージェントが内部で Grep/Glob を使って調査しました。

**プロセス:**
- ツール呼び出し: Explore エージェント 1回（内部で Grep/Glob を約13回）
- 所要ターン数: **1ターン**

**結果:**
- 実装クラス 2件（Cart, Order）
- 参照ファイル 128件
- 用途の要約

テキストパターン `implements ItemHolderInterface` で正確にヒットし、`use Eccube\Entity\ItemHolderInterface` で参照箇所も網羅できました。

## LSP（phpactor）の場合

phpactor の CLI サブコマンドを使って調査しました。

**プロセス:**
1. phpactor の存在確認 + 設定ファイル読み込み
2. コマンド体系の把握（`--help`, `list`）
3. `references:class` で正しいサブコマンドを探る試行錯誤
4. `class:reflect` でインターフェース定義の取得
5. サブクラス検索で `index:search` の構文ミス → リトライ
6. Cart/Order の `references:class` で継承確認
7. 出力が大きすぎてファイル保存 → 再読み込み

- 所要ターン数: **約10ターン・20回以上のツール呼び出し**

**結果:**
- 実装クラス 2件（Cart, Order）
- 参照 144件（行番号・カラムオフセット付き）
- 用途の要約

## 比較まとめ

| 項目 | Grep 検索 | LSP (phpactor) |
|---|---|---|
| ターン数 | 1 | 約10 |
| ツール呼び出し | 約13回 | 20回以上 |
| 実装クラス検出 | 2件 | 2件 |
| 参照件数 | 128件 | 144件 |
| 精度 | 十分 | カラムオフセットまで正確 |
| 試行錯誤 | なし | CLI構文の把握に複数回 |

## なぜ LSP の方が非効率だったのか

### 1. CLI インターフェースのオーバーヘッド

phpactor は本来 LSP サーバー（エディタ内で常駐し JSON-RPC で通信）として設計されています。CLI サブコマンド経由で使うと、毎回インデックスのロードが発生し、コマンド体系の把握にも試行錯誤が必要でした。

### 2. `implements` の検出にはテキスト検索で十分

`implements ItemHolderInterface` は構文的に明確なパターンなので、Grep で正確に見つかります。LSP の型解析能力（間接的な実装やジェネリクスの解決など）が活きる場面ではありませんでした。

### 3. 参照一覧もテキスト検索で同等の結果が出る

`use Eccube\Entity\ItemHolderInterface` や型ヒントの `ItemHolderInterface` は名前が一意なので、Grep で十分な精度が得られます。

## LSP が有利になるケース

逆に LSP が有効なのは以下のような場合です。

- **同名クラスが複数の名前空間に存在する**場合（テキスト検索だと誤検出する）
- **間接的な実装の追跡**（A implements B, C extends A → C も B の実装）が深い場合
- **メソッドのオーバーライド元の特定**など、型階層の解決が必要な場合
- **エディタ内でリアルタイムに使う**場合（常駐なのでインデックスロードが1回で済む）

## 結論

今回の `ItemHolderInterface` は名前が一意で、実装が Cart/Order の2クラスのみ、継承階層も浅いため、LSP の強みが発揮される条件ではありませんでした。

Claude Code でのコードベース調査においては、**まず Grep/Glob ベースの検索で十分かを検討し、型階層が複雑な場合や名前の衝突がある場合に LSP を使う**のが効率的です。

LSP は「常駐して使う」ことで真価を発揮するツールであり、CLI から都度呼び出すユースケースとは相性が良くないという点も重要な学びでした。
