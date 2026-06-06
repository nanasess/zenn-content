---
title: "【初心者向け】EC-CUBEで実現したいことを AI にやらせてみる会 Part2 | EC-CUBE名古屋 vol.122"
emoji: "🔍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["eccube","ClaudeCode","playwright"]
published: true
---
*この記事は「EC-CUBEで実現したいことを AI にやらせてみる会 Part2 | EC-CUBE名古屋 vol.122」の登壇資料です*

:::message alert
試行錯誤の内容をそのまま掲載しています。不具合のあるプログラムや、その修正過程を含みますので、参考にする場合はくれぐれもご注意ください。
:::

## 自己紹介

- 名前: 大河内健太郎(@nanasess) 年齢: 49才
- 出身: 愛知県西尾市一色町
- 在住: 大阪市(四天王寺の近く)
- 前職:寿司屋の板前
- 資格: 調理師・ふぐ処理師
- EC-CUBE コミッター・公式エバンジェリスト・名古屋ユーザーグループ

## 本日のテーマ

前回 [vol.121](https://zenn.dev/nanasess/articles/ai-customize-eccube-vol121) では、参加者から集まった**11個のお題**を Claude Code で並行実装し、**コード生成は11/11完了**したものの、勉強会の時間内に**動作確認まで終わったのは5/11**でした。

今回 vol.122 は、その**動作確認の続き**です。テーマを一言でいうと——

> **AIが書いたコードを、人間が実際に動かして確認する**

前回の記事の所感にこう書きました。

> AIは「コードを書く」ことは非常に得意ですが、「EC-CUBE 4 の実行環境で正しく動作させる」ためには、マイグレーション実行、プロキシ再生成、キャッシュクリア、PHP バージョン互換性など、人間側の知識と確認が必要でした。

今回はまさにこの「動作確認」を進めた結果、**AIが見落としていた実行時バグが次々と見つかりました**。本記事はその記録です。

### 前回（vol.121）との違い

|          | vol.121                        | vol.122                                          |
|----------|--------------------------------|--------------------------------------------------|
| 主目的   | 11お題の**実装**（コード生成） | 残りお題の**動作確認**とバグ修正                 |
| AIモデル | Claude Opus 4.6                | **Claude Opus 4.8（1M context）**                |
| 検証方法 | 手動                           | **playwright-cli によるブラウザ自動操作** + 手動 |

Opus 4.6 → 4.8 は2世代分の進化で、長時間の自律的なコーディングやバグ発見の精度が向上しています。今回のように「エラーを追い込む」「フローを何度も回す」といった作業に効きました。

## 環境

- Claude Code (Claude Opus 4.8)
- EC-CUBE 4.3系（Symfony 6.4 ベース）
- [git-worktree-manager](https://github.com/nanasess/git-worktree-manager) でお題ごとに分離した worktree（前回のものを継続利用）
- [playwright-cli](https://www.npmjs.com/package/@playwright/cli) でブラウザ操作を自動化し、画面表示やDB値を検証

## 進め方

1. お題ごとの worktree でビルトインサーバーを起動
2. 必要に応じてマイグレーション実行・プロキシ再生成・キャッシュクリア
3. **人間（＋playwright-cli）が実際に画面を操作して動作確認**
4. **不具合を見つけたら Claude Code に原因調査・修正させる**
5. 修正後に再検証し、問題なければコミット

ポイントは **「動作確認は人間（とブラウザ自動操作）、バグ修正はAI」** という役割分担です。

## 本日の動作確認結果

| # | お題 | ブランチ | 本日の結果 |
|---|---|---|---|
| 2 | 位置情報(都道府県)表示 | [geolocation](https://github.com/nanasess/ec-cube/tree/geolocation) | 📝 コードレビューのみ（外部サービス依存） |
| 3 | 管理画面の在庫数量検索 | [stock-search](https://github.com/nanasess/ec-cube/tree/stock-search) | ✅ 確認OK（バグなし） |
| 8 | 売上集計を発送日基準に | [sales-report](https://github.com/nanasess/ec-cube/tree/sales-report) | ✅ **SQLite対応バグを修正**して確認 |
| 9 | セール価格の期間限定設定 | [sale-price](https://github.com/nanasess/ec-cube/tree/sale-price) | ✅ **3件のバグを修正**して確認 |
| 11 | 名入れ機能 | [naire-engraving](https://github.com/nanasess/ec-cube/tree/naire-engraving) | ✅ **2件のバグを修正＋再設計**して確認 |

前回確認済みの 1, 5, 6, 7, 10 と合わせて、**動作確認 9/11 + コードレビュー1** まで進みました（残りは紹介クーポンのみ）。

---

## 見つかったバグと修正

ここからが本題です。「コードは生成できている」のに「実環境で動かすと落ちる」——そのギャップを具体的に見ていきます。

### お題3: 在庫数量検索（バグなし）

管理画面の商品検索に在庫数量の範囲指定を追加する機能。playwright-cli で管理画面にログインし、DBから算出した期待値（在庫>=900は5件、在庫0〜0は1件…）と画面の検索結果件数を突き合わせ、**4/4 すべて一致**。FormExtension + QueryCustomizer のシンプルな実装で、バグはありませんでした。

### お題8: 売上集計（SQLite方言バグ）

発送日基準で売上を集計する機能。動作確認のために発送日データを登録したところ——

**バグ①: SQLite に `DATE_FORMAT` 関数が無い**

```php
// AIが生成したコード（MySQL前提）
'monthly' => "DATE_FORMAT(sub.report_date, '%Y-%m')",
```

EC-CUBE は PostgreSQL / MySQL / SQLite をサポートしますが、生成コードは PostgreSQL(`TO_CHAR`) と MySQL(`DATE_FORMAT`) しか考慮しておらず、**SQLite だと集計画面が500エラー**になりました。`strftime` 分岐を追加して修正。

```php
if ($platform instanceof SqlitePlatform) {
    return match ($unit) {
        'monthly' => "strftime('%Y-%m', sub.report_date)",
        'yearly'  => "strftime('%Y', sub.report_date)",
        default   => "strftime('%Y-%m-%d', sub.report_date)",
    };
}
```

**おまけ: 公式プラグインも発送日基準にカスタマイズ**

公式の「[売上集計プラグイン (SalesReport42)](https://www.ec-cube.net/products/detail.php?product_id=1996)」は受注日基準です。これを発送日基準に変えるカスタマイズも実施しました。`getData()` の絞り込みを `order_date` から、発送日を持つ出荷を EXISTS で判定する形に変更し、`convertByTerm()` のバケットも発送日基準に。受注日基準と発送日基準で月別分布が明確に変わることを確認できました。

### お題11: 名入れ機能（フォーム送信バグ＋設計の作り直し）

商品詳細で名入れテキストを入力し、カート→注文確認→受注詳細まで保持・表示する機能。動作確認すると、**カートに名入れが表示されません**でした。

**バグ②: AddCartType のPOST名を取り違えていた**

```php
// AIが生成したコード
if (isset($addCartData['add_cart']['ProductClass'])) {  // ← 常に null
    $productClassId = $addCartData['add_cart']['ProductClass'];
}
```

EC-CUBE の `AddCartType` はブロックプレフィックスが無く、`ProductClass` は**ルート名**でPOSTされます（`add_cart[ProductClass]` ではない）。そのため `product_class_id` が取れず、名入れがセッションに保存されていませんでした。

```php
// 修正
$productClassId = $request->request->get('ProductClass');
```

**設計の作り直し: セッション + 別テーブル → OrderItem列**

修正後、カートには表示されましたが、**注文確認画面では表示されません**でした。原因は、名入れを `NaireInfo` という別テーブル（OrderItemとOneToOne）に**注文確定時に**永続化していたため、確定前の確認画面ではまだ存在しないこと。

参加者から「セッションではなく OrderItem の追加カラムに持たせた方がよいのでは」という提案をいただき、設計を作り直しました。

- 別テーブル `NaireInfo` を廃止し、`OrderItem.naire_text` カラムを追加
- `ItemHolderPreprocessor` でセッションの名入れを**確定前に** OrderItem へコピー（→ 確認画面でも表示）
- 受注詳細は `TemplateEvent` でソース注入して表示

これで「カート→注文確認→受注詳細」すべてで表示されるようになり、別テーブルも不要になりました（実装も -440行とスリムに）。

### お題9: セール価格（バグ3連発）

カテゴリ単位で期間限定セール価格を適用する機能。前回時点で「services.yamlのDI設定に問題が残存」とされていたお題ですが、実際に触ると**DIではなく別の3つのバグ**が見つかりました。

**バグ③: セール編集画面がシステムエラー**

```
Unable to transform data for property path "end_date": Expected a \DateTimeImmutable.
```

フォームの日付が `input='datetime_immutable'` なのに、エンティティのカラムは `datetimetz`（可変 `\DateTime`）。既存セールの**編集時**に型変換で落ちていました（新規作成はnullなので顕在化せず）。`input='datetime'` に揃えて修正。

**バグ④: 取り消し線がセール価格と同じ値になる**

商品詳細で `~~￥2,772~~ ￥2,772 SALE` と、取り消し線もセール価格になっていました。原因は、`PriceChangeValidator` との整合のために postLoad で `price02IncTax` をセール価格に**上書き**しており、表示側の取り消し線もその上書き後の値を読んでいたこと。上書き前の元価格を退避して表示で使うよう修正（`~~￥3,080~~ ￥2,772 SALE` に）。

**バグ⑤: 注文確認で「販売価格が変更されました」エラーで進めない**

これは参加者の手動確認で発見。注文フローの `PriceChangeValidator` は OrderItem を `price02`(**税抜・実カラム**)と比較しますが、上書きされるのは計算値 `price02_inc_tax` のみで `price02` は元のまま。そのため Preprocessor が設定したセール価格(税抜2,520)と `price02`(2,800)が一致せず、**確認画面の再検証で必ず弾かれて**いました。

`price02` は実カラムのため上書きすると永続化リスクがあり触れません。そこでセール対応の `SaleAwarePriceChangeValidator` を作り、**セール中のOrderItemは価格変更検知をスキップ**（セール価格は Preprocessor が毎回設定するため安全）するよう差し替えました。

注文確定まで通り、`OrderItem` に `price=2520 / original_price=2800 / sale_config_id=1` が記録されることも確認しました。

**確認のみ（修正なし）: 管理画面でのセール商品追加**

「セール期間終了後、管理画面でセール期間中の受注にセール対象商品を追加すると、通常価格で登録されてしまうか？」という鋭い質問もありました。コードを追うと、

- 管理受注編集では postLoad / Preprocessor がセール適用をスキップ（管理者の裁量を尊重する設計）
- 商品追加時の価格はコアの `OrderItemType` が `ProductClass.getPrice02()`（通常価格）から設定

→ **追加した明細は通常価格**になります（既存明細はセール価格のまま）。期待は「セール価格で追加」でしたが、これは設計上の挙動。今回は確認のみとしました（将来の拡張候補）。

## 番外編: 4系の在庫検索を2系にも移植する

お題3「在庫数量検索」は EC-CUBE 4系での実装でしたが、「**同じ機能を 2系にも欲しい**」という要望が出ました。4系には商品一覧検索に標準で**在庫有無（あり/なし）**の検索条件がありますが、2系の商品マスター検索には在庫関連の条件が一切ありません。

そこで、

- 4系相当の**在庫有無（あり/なし）検索**を 2系に追加
- さらに 2系独自要望として**在庫数量の上限/下限**での絞り込みも追加

を Claude Code に実装させました。**参照元（4系）と実装先（2系）の2つのコードベースを同時に渡せる**のが、AIによるバージョン間移植の強みです。

### 進め方

今回は「動作確認」ではなく「**移植元の仕様を正確に読み取って実装先に翻訳する**」タスクなので、まず両コードベースを並行調査させました。

1. 4系（`~/git-repos/ec-cube`）の在庫有無検索の実装を調査
   - フォーム `SearchProductType`、`ProductRepository::getQueryBuilderBySearchDataForAdmin()` の判定ロジック
2. 2系（`~/git-repos/ec-cube2`）の商品検索の実装パターンを調査
   - `LC_Page_Admin_Products` の「`lfInitParam()` でパラメータ定義 → `index.tpl` でUI → `buildQuery()` でWHERE句構築」という確立されたパターン
3. 判定ロジックを 2系の流儀（`dtb_products_class` へのサブクエリ）に翻訳

### 4系の判定ロジック

4系では商品規格テーブル `dtb_product_class` の `stock` と `stock_unlimited`（無制限フラグ）で在庫を判定しています。

```php
// 4系: ProductRepository::getQueryBuilderBySearchDataForAdmin()
case [ProductStock::IN_STOCK]:   // 在庫あり
    $qb->andWhere('pc.stock_unlimited = true OR pc.stock > 0');
    break;
case [ProductStock::OUT_OF_STOCK]: // 在庫なし
    $qb->andWhere('pc.stock_unlimited = false AND pc.stock <= 0');
    break;
// 両方選択時は全件該当するので条件に含めない
```

### 2系への翻訳

2系も同じ `dtb_products_class.stock` / `stock_unlimited` を持つので、判定式はそのまま流用できます。2系では既存の `search_product_code` 検索と同じく **`product_id IN (SELECT ... FROM dtb_products_class ...)`** のサブクエリで表現しました（複数規格は「いずれかが条件を満たせばヒット」）。

```php
// 2系: LC_Page_Admin_Products::buildQuery() に追加した case
// 在庫有無
case 'search_stock':
    $arrStock = (array) $objFormParam->getValue($key);
    $hasIn = in_array('1', $arrStock);  // あり
    $hasOut = in_array('2', $arrStock); // なし
    if ($hasIn && !$hasOut) {
        $where .= ' AND product_id IN (SELECT product_id FROM dtb_products_class WHERE del_flg = 0 AND (stock_unlimited = 1 OR stock > 0))';
    } elseif ($hasOut && !$hasIn) {
        $where .= ' AND product_id IN (SELECT product_id FROM dtb_products_class WHERE del_flg = 0 AND stock_unlimited = 0 AND (stock <= 0 OR stock IS NULL))';
    }
    break;
// 在庫数量(下限) … 無制限商品はヒットさせる
case 'search_stock_min':
    $where .= ' AND product_id IN (SELECT product_id FROM dtb_products_class WHERE del_flg = 0 AND (stock_unlimited = 1 OR stock >= ?))';
    $arrValues[] = $objFormParam->getValue($key);
    break;
// 在庫数量(上限) … 無制限商品は除外
case 'search_stock_max':
    $where .= ' AND product_id IN (SELECT product_id FROM dtb_products_class WHERE del_flg = 0 AND stock_unlimited = 0 AND stock <= ?)';
    $arrValues[] = $objFormParam->getValue($key);
    break;
```

ポイントは2つ。

- **`stock IS NULL` の補完**: 2系の `stock` カラムは nullable なので、「なし」判定に `stock <= 0 OR stock IS NULL` を追加（4系の Doctrine 経由とは異なり生SQLのため）。
- **在庫無制限の扱い**（在庫数量検索）: 無制限商品は「在庫が無限にある」とみなし、**下限指定時はヒット／上限指定時は除外**。

### 変更ファイル

| ファイル | 変更内容 |
|---|---|
| `data/class/pages/admin/products/LC_Page_Admin_Products.php` | `init()` に在庫有無の選択肢、`lfInitParam()` にパラメータ3つ、`buildQuery()` に case 3つ |
| `data/Smarty/templates/admin/products/index.tpl` | 「在庫有無」チェックボックスと「在庫数量」の範囲入力を検索フォームに追加 |

2系の確立された追加パターン（パラメータ定義 → テンプレート → WHERE句）に完全に乗せたため、`alldtlSQL()` や一覧表示ロジックには一切手を入れずに済みました。AIに「既存の `search_product_code` と同じパターンで」と方向づけられたのが効いています。

実装は [nanasess/ec-cube2 の stock-search ブランチ](https://github.com/nanasess/ec-cube2/tree/stock-search) にあります。

:::message
2系は PHPUnit がすぐに動かない環境だったため、本件は管理画面での手動確認のみで進めました。MySQL / PostgreSQL いずれでも動く SQL（`ILIKE` 等の方言に依存しない）にしてあります。
:::

---

## まとめ

### 数字で振り返る

- 本日の動作確認: **未確認だった4お題 + コードレビュー1**（累計 9/11 + レビュー1）
- 見つけて修正したバグ: **6件**（SQLite方言 / フォーム送信 / 別テーブル設計 / 日付型 / 取り消し線 / 価格再検証）
- 確認のみの設計課題: **1件**（管理画面でのセール商品追加）

### バグの傾向

見つかったバグには共通点がありました。**どれも「コードを読むだけでは気づきにくく、実行して初めて落ちる」類のもの**です。

- DB方言の差（SQLite に `DATE_FORMAT` が無い）
- フレームワークの仕様（`AddCartType` のPOST名）
- 永続層と表示層の型不整合（`datetime_immutable` vs `datetimetz`）
- 値を共有するフィールドの副作用（`price02IncTax` 上書きが表示に波及）
- 実行順序に依存する不具合（PurchaseFlow の validator → preprocessor 順）

AIはこれらを「もっともらしく」書きますが、**実際に動かすまで誤りが表面化しません**。逆に言えば、**動作確認さえすれば、原因の特定と修正はAIが非常に得意**でした。エラーメッセージやログを渡すと、ほぼ一発で原因（DB方言、POST名の取り違え、型不整合など）を言い当て、修正案を出してきます。

### 所感

vol.121 で「AIは実装、人間はアーキテクチャ理解と動作確認」と書きましたが、vol.122 でそれがより具体的になりました。

> **人間（とブラウザ自動操作）が動かして不具合を見つけ、AIが原因を特定して直す。**

この往復のサイクルが非常に高速で、4つのお題で6件のバグを潰しながら、注文フローの最後まで通すところまで到達できました。playwright-cli でブラウザ操作とDB検証を自動化したことで、「実際に画面で確認 → 期待値と突き合わせ」の負担も大きく下がりました。

「AIにコードを書かせる」だけでなく、**「AIが書いたコードを、人間が確認し、AIに直させる」**——この協業の形が、実用的なカスタマイズへの近道だと改めて感じた回でした。

なお、この記事自体も Claude Code に作成・コミットさせています。

## 参考リンク

- [前回(vol.121)の記事](https://zenn.dev/nanasess/articles/ai-customize-eccube-vol121)
- [vol.113 の記事（EC-CUBE 2系）](https://zenn.dev/nanasess/articles/ai-customize-eccube)
- [EC-CUBE 4系 リポジトリ](https://github.com/EC-CUBE/ec-cube)
- [git-worktree-manager](https://github.com/nanasess/git-worktree-manager)
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
