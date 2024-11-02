---
title: "【初心者向け】タグマネージャーで GA4 のEコマース設定をしよう | EC-CUBE名古屋 vol.109"
emoji: "🔪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ga4", "gtm", "ecommerece", "eccube"]
published: false
---
*この記事は「【初心者向け】タグマネージャーで GA4 のEコマース設定をしよう | EC-CUBE名古屋 vol.109」の登壇資料です*

## GA4 と Eコマースイベント

https://support.google.com/analytics/answer/12200568?hl=ja&ref_topic=12270146&sjid=693972572606023965-AP

## Eコマースイベントの設定方法

- Google タグ
  - 難易度は低いが汎用性、拡張性も低い
- タグマネージャー
  - 難易度は高いが汎用性、拡張性高い

ここではタグマネージャーを使ってみます

### GitHub Copilot の回答

#### Googleタグ

##### メリット

- シンプル: 基本的なトラッキングニーズには設定が簡単。
- 直接統合: 中間システムを必要とせず、GA4と直接統合。
- パフォーマンス: タグマネージャーに比べてオーバーヘッドが少なく、若干のパフォーマンス向上。

##### デメリット

- 柔軟性に欠ける: 複雑なトラッキングやカスタマイズには不向き。
- 拡張性: トラッキング要件が増えると管理が難しくなる。
- バージョン管理: 限られたバージョン管理とテスト環境。

#### タグマネージャー

##### メリット

- 柔軟性: 複雑なトラッキング設定にも対応可能。
- 一元管理: すべてのタグ、トリガー、変数を一元的に管理。
- バージョン管理: 内蔵のバージョン管理とデバッグツール。
- 拡張性: トラッキングニーズが増えても対応しやすい。
- サードパーティ統合: 幅広いサードパーティタグや統合をサポート。

##### デメリット

- 複雑性: 設定と管理が複雑で、特に初心者には難しい。
- 学習曲線: 多様な機能とオプションがあり、習得に時間がかかる。
- パフォーマンスオーバーヘッド: タグマネージャーの追加によりページ読み込み時間が若干増加。

## タグマネージャーの設定手順

https://developers.google.com/analytics/devguides/collection/ga4/ecommerce?client_type=gtm&hl=ja

1. タグマネージャーでイベントを設定する
2. イベントのトリガーを設定する
3. タグの実装をする

1 と 2 はタグマネージャーの管理画面で設定します。
3 は PHP と JavaScript を駆使して、 データベースから情報を取り出し、データレイヤーに渡します。

Eコマーストラッキングでは以下のようなイベントの実装が必要です。

- リストからアイテムを選択する
- アイテム詳細の閲覧
- ショッピング カートへの商品の追加または削除
- 購入手続きの開始
- 購入または払い戻し
- プロモーションの適用

https://developers.google.com/analytics/devguides/collection/ga4/ecommerce?client_type=gtm&hl=ja#implementation

### 実装コードのサンプル

ちゃんとやろうとすると強烈に難易度が高い

EC-CUBE3系
https://amidaike.hatenablog.com/entry/2021/12/08/000557

EC-CUBE4系
https://amidaike.hatenablog.com/entry/2021/12/04/004709

#### EC-CUBE2系の商品一覧の実装例

*申し訳ございません. まだ動作確認ができていません*


- 商品一覧を出力する Smarty 変数は既にある
- しかし、各商品の所属するカテゴリ情報は保持していないため、新たに取得する必要がある(PHPの実装が必要。性能問題もある)

PHPの実装
``` php
foreach ($arrProducts as &$arrProduct) {
    $arrProduct['arrRelativeCat'] = SC_Helper_DB_Ex::sfGetMultiCatTree($arrProduct['$product_id']);
}
```

Smarty の実装

**カテゴリが入れ子の section になっているので、おそらくこの実装ではうまく動作しません**
**商品ごとのループだと price が取得できないため、本来は SKU (商品規格)ごとに出力する必要があると思われます**

```Smarty
dataLayer.push({ ecommerce: null });  // Clear the previous ecommerce object.
dataLayer.push({
  event: "view_item_list",

  ecommerce: {
  <!--{foreach from=$arrProducts item=arrProduct name=arrProducts}-->
     {
      item_id: "<!--{$arrProduct.product_id|u}-->",
      item_name: "<!--{$arrProduct.name|h}-->",
      index: <!--{$smarty.foreach.arrProducts.index}-->,
      <!--{section name=r loop=$arrProduct.arrRelativeCat}-->
          <!--{section name=s loop=$arrProduct.arrRelativeCat[r]}-->
              <!--{if $smarty.section.s.index == 0}}-->
                  item_category: "<!--{$arrRelativeCat[r][s].category_name|h}-->",
              <!--{else}-->
                  item_category<!--{$smarty.section.s.iteration}-->: "<!--{$arrRelativeCat[r][s].category_name|h}-->",
              <!--{/if}-->
          <!--{/section}-->
      <!--{/section}-->
      price: <!--{$arrProduct.price02_max_inctax|n2s|default:0}-->,
      quantity: 1
    }<!--{if !$smarty.foreach.arrProducts.last}-->,<!--{/if}-->
<!--{/foreach}-->

