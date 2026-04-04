---
title: "EC-CUBEで実現したいことをAIにやらせてみる会 | EC-CUBE名古屋 vol.121"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["eccube","ClaudeCode"]
published: true
---
*この記事は「【初心者向け】EC-CUBEで実現したいことをAIにやらせてみる会 | EC-CUBE名古屋 vol.121」の登壇資料です*

:::message alert
試行錯誤の内容をそのまま掲載しています。不具合のあるプログラムも含まれていますので、参考にする場合はくれぐれもご注意ください。
:::

## 自己紹介

- 名前: 大河内健太郎(@nanasess) 年齢: 49才
- 出身: 愛知県西尾市一色町
- 在住: 大阪市(四天王寺の近く)
- 前職:寿司屋の板前
- 資格: 調理師・ふぐ処理師
- EC-CUBE コミッター・公式エバンジェリスト・名古屋ユーザーグループ
- 最近のマイブーム: ここ1年自分でコード書かなくなりました

## 本日のテーマ

昨年の vol.113 に続き、参加者が持ち寄った「EC-CUBEで実現したいこと」をAIを活用して実装してみます！

前回は Cline + Claude 3.7 Sonnet で試しましたが、今回は **Claude Code + Claude Opus 4.6** を使って試してみたいと思います。
AIの進化に伴い、より高度な実装が期待できるはず...！

### 前回（vol.113）との違い

| | vol.113 (2025年) | vol.121 (2026年) |
|---|---|---|
| EC-CUBE | 2系 | **4系（Symfony 6.4 ベース）** |
| AIツール | Cline + Claude 3.7 Sonnet | **Claude Code + Claude Opus 4.6** |
| カスタマイズ方式 | 直接ファイル編集 | **Customize ディレクトリ / プラグイン** |

## 環境

- Claude Code (Claude Opus 4.6)
- EC-CUBE 4.3系（Symfony 6.4 ベース）
- Docker Compose
- [git-worktree-manager](https://github.com/nanasess/git-worktree-manager) でお題ごとにworktreeを分離

## 進め方

参加者8名から11個のお題を募集し、以下の手順で進めました。

1. 11個のお題すべてについて **並行して実装計画を作成**（Agent を11個同時起動）
2. 各お題ごとに **git-worktree-manager で独立した worktree を作成**
3. 各 worktree の CLAUDE.md に実装計画を記載
4. **11個のお題を同時に並行実装**（Agent を11個同時起動）
5. レート制限で中断したものは再実行
6. 全 worktree で `composer install` + `bin/console eccube:install` を並行実行
7. 動作確認

## お題一覧

参加者から集まった11個のお題です。

| # | お題 | worktree | 動作確認 |
|---|---|---|---|
| 1 | フロント画面のサジェスト機能 | `suggest-feature` | **確認済** |
| 2 | ユーザーの位置情報(都道府県)を表示 | `geolocation` | 未確認 |
| 3 | 管理画面の商品検索に在庫数量を追加 | `stock-search` | 未確認 |
| 4 | 紹介クーポン機能 | `referral-coupon` | 未確認 |
| 5 | 品切れ表示を欠品と廃盤に分離 | `out-of-stock-type` | **確認済** |
| 6 | 特定のお客様専用URLの商品ページ | `private-product` | **確認済** |
| 7 | 受注リストを注文番号でソート | `order-sort` | **確認済** |
| 8 | 売上集計を発送日基準にする | `sales-report` | 未確認 |
| 9 | セール価格の期間限定価格設定 | `sale-price` | エラーあり |
| 10 | Aの商品をカートに入れたらBが割引 | `cross-product-discount` | **確認済** |
| 11 | 名入れ機能 | `naire-engraving` | 未確認 |

## 動作確認できたお題

### お題1: フロント画面のサジェスト機能

**やりたいこと**: ヘッダーの検索欄にキーワードを入力すると、商品候補がドロップダウンで表示される。

**実装内容**:
- `app/Customize/Controller/ProductSuggestController.php` - JSON APIエンドポイント
- `html/user_data/assets/js/customize.js` - jQuery Ajaxオートコンプリート（デバウンス、キーボード操作対応）
- `html/user_data/assets/css/customize.css` - サジェストUI用CSS（レスポンシブ対応）
- `app/template/default/Block/search_product.twig` - テンプレートオーバーライド

既存の `ProductRepository::getQueryBuilderBySearchData()` を再利用し、最大10件の候補を返すAPIを作成。コアファイルの変更なし。

### お題5: 品切れ表示を欠品と廃盤に分離

**やりたいこと**: 「ただいま品切れ中です。」の一律表示を、「欠品（一時的）」と「廃盤（恒久的）」に分けたい。

**実装内容**:
- `app/Customize/Entity/ProductClassTrait.php` - `out_of_stock_type` カラム追加
- `app/Customize/Entity/ProductTrait.php` - 商品レベルの品切れ種別集約
- `app/Customize/Form/Extension/ProductClassExtension.php` - 管理画面フォーム拡張
- `app/Customize/EventListener/AdminProductEditSubscriber.php` - 管理画面テンプレートへの注入
- `app/template/default/Product/detail.twig` / `list.twig` - フロント表示テンプレート
- マイグレーション、翻訳ファイル

**つまづいたポイント**:
- PHP 8.1 では trait の定数に `TraitName::CONSTANT` で直接アクセスできない（PHP 8.2以降の機能）。定数値をリテラルに変更して対応
- FormExtension でフィールドを追加しても、テンプレートに `form_row` がないと画面に表示されない。EventSubscriber でテンプレートに注入する方式で対応
- `bin/console eccube:generate:proxies` を実行しないと Entity Trait の getter/setter が認識されない

### お題6: 特定のお客様専用URLの商品ページ（限定公開）

**やりたいこと**: 非公開商品を秘密トークン付きURLで特定顧客だけに公開したい。

**実装内容**:
- `app/Customize/Entity/PrivateLink.php` - トークン管理エンティティ
- `app/Customize/Controller/PrivateLinkController.php` - フロント（トークン検証 + 商品表示 + カート追加）
- `app/Customize/Controller/Admin/PrivateLinkController.php` - 管理画面（リンク発行/管理）
- `app/Customize/EventSubscriber/AdminProductEventSubscriber.php` - 商品編集画面への導線
- マイグレーション、テンプレート

トークンは `bin2hex(random_bytes(32))` で64文字のランダム文字列を生成。既存の `checkVisibility()` をバイパスする独自コントローラで実装。

### お題7: 受注リストを注文番号でソート

**やりたいこと**: 管理画面の受注一覧で注文番号でソートしたい。

**実装内容（最小限の2ファイル変更）**:
- `src/Eccube/Repository/OrderRepository.php` - COLUMNS定数に `'order_no' => 'o.order_no'` を追加
- `src/Eccube/Resource/template/admin/Order/index.twig` - テーブルヘッダにソートリンク追加

11個の中で最もシンプルな変更。既存のソート基盤が汎用的に作られているため、定数追加とUIリンク追加のみで完結。

### お題10: Aの商品をカートに入れたらBが割引（セット割引）

**やりたいこと**: 花嫁衣装をカートに入れると紋付のレンタルが半額になるような機能。

**実装内容**:
- `app/Customize/Entity/CrossProductDiscount.php` - 割引ルールエンティティ
- `app/Customize/Service/PurchaseFlow/Processor/CrossProductDiscountProcessor.php` - DiscountProcessor（cart + shopping 両フローに登録）
- `app/Customize/Controller/Admin/CrossProductDiscountController.php` - 管理画面CRUD
- マイグレーション、フォーム、テンプレート

PurchaseFlow の `DiscountProcessor` を実装し、カート内の商品組み合わせをリアルタイムに判定。トリガー商品をカートから削除すると割引も自動解除される。

## 動作確認できなかったお題

時間切れで動作確認まで至らなかったものの、コードは全て実装済みです。

- **位置情報表示**: GeoIP2ライブラリ + EventSubscriber + 送料見積もりAPI
- **在庫数量検索**: FormExtension + QueryCustomizer（コア変更不要）
- **紹介クーポン**: 最大規模。エンティティ〜PurchaseFlow〜返品対応〜メール〜マイページまで
- **売上集計**: 発送日基準の集計ページ + Chart.jsグラフ + CSV出力
- **セール価格**: services.yaml の DI 設定に問題が残存（要修正）
- **名入れ機能**: CartItem の `__sleep()` 制約を回避するセッション管理方式

## まとめ

### 数字で振り返る

- 参加者からのお題: **11個**
- 並行実装エージェント: **最大11個同時**
- 実装完了: **11/11**（コード生成）
- 動作確認: **5/11**（約4時間の勉強会内）
- 途中のレート制限: **2回**（再実行で対応）

### 前回（vol.113）との比較

前回は EC-CUBE 2系で3つのお題に取り組みました。今回は EC-CUBE 4系（より複雑なアーキテクチャ）で11個のお題に同時に取り組み、5個の動作確認まで完了できました。

**Claude Code の強み**:
- git-worktree-manager と組み合わせることで、11個の独立した実装を並行管理できた
- Agent 機能で複数のお題を同時に実装計画 → 同時にコード生成
- EC-CUBE 4 の Customize ディレクトリ、FormExtension、PurchaseFlow、EventSubscriber 等のアーキテクチャを理解した上でのコード生成

**つまづきポイント**:
- PHP バージョン互換性（trait 定数は PHP 8.2 以降）をAIが考慮できていなかった
- FormExtension でフィールド追加してもテンプレートへの表示は別途必要
- services.yaml での DI 設定（オートワイヤリングとの兼ね合い）
- `eccube:generate:proxies` の実行忘れ

### 所感

AIは「コードを書く」ことは非常に得意ですが、「EC-CUBE 4 の実行環境で正しく動作させる」ためには、マイグレーション実行、プロキシ再生成、キャッシュクリア、PHP バージョン互換性など、人間側の知識と確認が必要でした。AIと人間の協業がうまくいくポイントは、AIに実装を任せつつ、人間がアーキテクチャの理解と動作確認を担うことだと感じました。

## 参考リンク

- [EC-CUBE名古屋 vol.121 イベントページ](https://ec-cube-nagoya.doorkeeper.jp/events/195941)
- [前回(vol.113)の記事](https://zenn.dev/nanasess/articles/ai-customize-eccube)
- [EC-CUBE 4系 リポジトリ](https://github.com/EC-CUBE/ec-cube)
- [git-worktree-manager](https://github.com/nanasess/git-worktree-manager)
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
