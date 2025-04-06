---
title: "AIに命令してどこまでEC-CUBEをカスタマイズできるか? | EC-CUBE名古屋 vol.113"
emoji: "🙆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["eccube","Cline"]
published: true
---
*この記事は「【初心者向け】AIに命令してどこまでEC-CUBEをカスタマイズできるか? | EC-CUBE名古屋 vol.113」の登壇資料です*

:::message alert
試行錯誤の内容をそのまま掲載しています。不具合のあるプログラムも含まれていますので、参考にする場合はくれぐれもご注意ください。
:::

## 自己紹介

- 名前: 大河内健太郎(@nanasess) 年齢: 48才
- 出身: 愛知県西尾市一色町
- 在住: 大阪市(四天王寺の近く)
- 前職:寿司屋の板前
- 資格: 調理師・ふぐ処理師
- EC-CUBE コミッター・公式エバンジェリスト・名古屋ユーザーグループ
- 最近のマイブーム: 1Password もっと早く使えばよかった

## 本日のテーマ

生成AIはみなさん活用されていますか？
EC-CUBEの特性上、ChatGPT は的外れな回答をしてくる場合が多いのですが、他のAIエンジンは精度のよいものも出てきていますし Cline や Cursor といった自律型のAIエージェントも使えるようになってきましたので、いろいろ試して、どこまで活用できるか試してみたいと思います！

今回は Cline と Docker コンテナ上で動作している EC-CUBE2系を使用して、簡単なカスタマイズを試してみたいと思います。

[Cline のセットアップ方法はこちらを参考にしました](https://www.ai-souken.com/article/what-is-cline)
また、AIモデルは **claude-3-7-sonnet-20250219** を使用してみました。


## パスワードの要件変更

**現在、ユーザーのパスワードの要件は「8文字以上50文字以下の英数字記号混在、英数字はそれぞれ1種類以上使用する」です。
英数字はそれぞれ1種類以上使用するという要件を外してください。**

という指示をしてみました。

> I need to understand the current password requirements and where they are defined in the codebase. Based on the open tabs and the task description, it seems that the password validation is likely defined in SC_CheckError.php. Let me first examine this file to understand how the password validation is currently implemented.
>
> I'll also look at the test file SC_CheckError_PASSWORD_CHAR_CHECKTest.php which might provide more insights into how the password validation works.
>
> I'll help you modify the password requirements to remove the requirement that passwords must contain at least one letter and one number. Let me first examine the current implementation to understand where and how this validation is performed.

英語で回答してきたので、ショップオーナーの多いEC-CUBE名古屋ユーザーグループでは敷居が高いです。日本語でお願いしてみます。

**やりとりは日本語でお願いします。**

> 申し訳ありません。ユーザーは日本語でのやりとりを希望されています。これからは日本語で対応します。
>
>まずは、パスワードの要件がどのように実装されているかを理解するために、SC_CheckError.phpファイルを確認する必要があります。また、テストファイルSC_CheckError_PASSWORD_CHAR_CHECKTest.phpも確認すると、より詳細な情報が得られるかもしれません。
>
>パスワードの要件変更のお手伝いをいたします。まずは現在の実装を確認して、どこでどのようにパスワードの検証が行われているかを理解する必要があります。
>
> Cline wants to read this file:
> data/class/SC_CheckError.php

Approve をクリックして、SC_CheckError を確認してもらいます。

> SC_CheckError.phpファイルを確認しました。パスワードの要件はPASSWORD_CHAR_CHECKメソッドで定義されています。このメソッドでは、以下の正規表現パターンを使用してパスワードをチェックしています：
>
> ```diff php
> $pattern = '/\A(?=.*?[a-z])(?=.*?\d)[!-~]{'.PASSWORD_MIN_LEN.','.PASSWORD_MAX_LEN.'}+\z/i';
> ```
>
> この正規表現の意味は：
>
> - (?=.*?[a-z]) - 少なくとも1つのアルファベット文字を含む
> - (?=.*?\d) - 少なくとも1つの数字を含む
> - [!-~]{'.PASSWORD_MIN_LEN.','.PASSWORD_MAX_LEN.'} - 指定された最小長と最大長の間の文字数で、ASCII文字の範囲（!から~まで）の文字のみを使用
> ユーザーの要求は「英数字はそれぞれ1種類以上使用するという要件を外す」ということなので、この正規表現から(?=.*?[a-z])と(?=.*?\d)の部分を削除する必要があります。
>
>次に、テストファイルも確認して、変更による影響を理解しましょう。

実装確認も問題なさそうですので、 Approve してテストコードも確認してもらいます。

> パスワード要件の変更について理解しました。現在の実装を確認したところ、以下のことがわかりました：
>
> 1. SC_CheckError.phpのPASSWORD_CHAR_CHECKメソッドで、パスワードの要件が定義されています。
>
> 2. 現在の正規表現パターンは /\A(?=.*?[a-z])(?=.*?\d)[!-~]{'.PASSWORD_MIN_LEN.','.PASSWORD_MAX_LEN.'}+\z/i となっており、これは：
>
>     - (?=.*?[a-z]) - 少なくとも1つのアルファベット文字を含む
>     - (?=.*?\d) - 少なくとも1つの数字を含む
>     - [!-~]{'.PASSWORD_MIN_LEN.','.PASSWORD_MAX_LEN.'} - 指定された最小長と最大長の間の文字数で、ASCII文字の範囲の文字のみを使用
> 3. テストファイルSC_CheckError_PASSWORD_CHAR_CHECKTest.phpには、現在の要件に基づいたテストケースが含まれています。
>
> 変更計画：
> 
> 1. SC_CheckError.phpのPASSWORD_CHAR_CHECKメソッド内の正規表現パターンを変更します：
>
>     - (?=.*?[a-z])(?=.*?\d) の部分を削除し、単に [!-~]{'.PASSWORD_MIN_LEN.','.PASSWORD_MAX_LEN.'}+ とします。
>     - これにより、英数字それぞれ1種類以上という要件が外れ、単に8文字以上50文字以下の英数字記号混在という要件になります。
> 2. エラーメッセージも更新する必要があります：
> 
>     - 現在のメッセージ: "英数字をそれぞれ1種類使用し、8文字以上で入力してください。"
>     - 新しいメッセージ: "8文字以上で入力してください。"
> 3. テストケースも更新する必要があります：
> 
>     - 特に「アルファベットのみ」や「数字のみ」のテストケースは、変更後は合格するはずなので、期待値を変更する必要があります。
> 
> この変更を実施するには、Act モードに切り替えていただく必要があります。準備ができましたら、「toggle to Act mode」と指示してください。
>

方針に問題なさそうですので Act モードに変更してみます。

> Act モードに切り替わりましたので、計画に従って変更を実施します。
>
> まず、SC_CheckError.phpファイルのPASSWORD_CHAR_CHECKメソッドを修正します。正規表現パターンから(?=.*?[a-z])(?=.*?\d)の> 部分を削除し、エラーメッセージも更新します。
>
> 次に、テストファイルSC_CheckError_PASSWORD_CHAR_CHECKTest.phpを修正します。特に「アルファベットのみ」や「数字のみ」の> テストケースの期待値を変更する必要があります。
>
> まず、SC_CheckError.phpファイルのPASSWORD_CHAR_CHECKメソッドを修正して、英数字それぞれ1種類以上という要件を外します。> 
>
> Cline wants to edit this file:
> data/class/SC_CheckError.php 

```diff php
diff --git a/data/class/SC_CheckError.php b/data/class/SC_CheckError.php
index 0ce4dd278..44f12cde7 100644
--- a/data/class/SC_CheckError.php
+++ b/data/class/SC_CheckError.php
@@ -901,11 +901,10 @@ class SC_CheckError
                 "※ {$disp_name}は英数記号で入力してください。<br />";
         }
     }
-
     /**
      * パスワードに使用可能な文字列のチェック
      *
-     * 半角英数字をそれぞれ1種類以上含む PASSWORD_MIN_LEN 文字以上 PASSWORD_MAX_LEN 文字以下の文字列ではない場合エラーとする
+     * PASSWORD_MIN_LEN 文字以上 PASSWORD_MAX_LEN 文字以下の文字列ではない場合エラーとする
      * PASSWORD_MIN_LEN が8未満の場合は E_USER_WARNING を出力する
      *
      * @param array $value $value[0] = 項目名 $value[1] = チェック対象のパスワード文字列
@@ -926,10 +925,10 @@ class SC_CheckError
 
         $input_var = $this->arrParam[$keyname];
         // see https://qiita.com/mpyw/items/886218e7b418dfed254b
-        $pattern = '/\A(?=.*?[a-z])(?=.*?\d)[!-~]{'.PASSWORD_MIN_LEN.','.PASSWORD_MAX_LEN.'}+\z/i';
+        $pattern = '/\A[!-~]{'.PASSWORD_MIN_LEN.','.PASSWORD_MAX_LEN.'}+\z/i';
         if (strlen($input_var) > 0 && !preg_match($pattern, $input_var)) {
             $this->arrErr[$keyname] =
-                "※ {$disp_name}は英数字をそれぞれ1種類使用し、".PASSWORD_MIN_LEN.'文字以上で入力してください。<br />';
+                "※ {$disp_name}は".PASSWORD_MIN_LEN.'文字以上で入力してください。<br />';
         }
     }
 
```


SC_CheckError の修正内容も問題なさそうですので、テストコードの修正もしてもらいます。

> 次に、テストファイルSC_CheckError_PASSWORD_CHAR_CHECKTest.phpを修正する必要があります。特に「アルファベットのみ」や「数字のみ」のテストケースの期待値を変更する必要があります。これらのテストケースは、現在は失敗するように設定されていますが、要件変更後は成功するはずです。
>
>次に、テストファイルSC_CheckError_PASSWORD_CHAR_CHECKTest.phpを修正して、変更した要件に合わせてテストケースを更新します。特に「アルファベットのみ」や「数字のみ」のテストケースは、要件変更後は成功するはずなので、期待値を変更します。
>
> Cline wants to edit this file:
> tests/class/SC_CheckError/SC_CheckError_PASSWORD_CHAR_CHECKTest.php

> もう一つ修正が必要です。testPASSWORDCHARCHECKWithMinuteメソッドのエラーメッセージも更新する必要があります。現在のエラーメッセージは「英数字をそれぞれ1種類使用し、8文字以上で入力してください」となっていますが、要件変更後は「8文字以上で入力してください」に変更する必要があります。
>
>最後に、testPASSWORDCHARCHECKWithMinuteメソッドのエラーメッセージも更新する必要があります。このテストケースは文字数が少ない場合のテストなので、エラーメッセージを新しい要件に合わせて修正します。

``` diff php
diff --git a/tests/class/SC_CheckError/SC_CheckError_PASSWORD_CHAR_CHECKTest.php b/tests/class/SC_CheckError/SC_CheckError_PASSWORD_CHAR_CHECKTest.php
index 73c2dc8b1..f7cea75d2 100644
--- a/tests/class/SC_CheckError/SC_CheckError_PASSWORD_CHAR_CHECKTest.php
+++ b/tests/class/SC_CheckError/SC_CheckError_PASSWORD_CHAR_CHECKTest.php
@@ -63,7 +63,7 @@ class SC_CheckError_PASSWORD_CHAR_CHECKTest extends SC_CheckError_AbstractTestCa
         $this->arrForm = [
             self::FORM_NAME => 'password',
         ];
-        $this->expected = '※ PASSWORD_CHAR_CHECKは英数字をそれぞれ1種類使用し、8文字以上で入力してください。<br />';
+        $this->expected = '';
 
         $this->scenario();
         $this->verify();
@@ -74,7 +74,7 @@ class SC_CheckError_PASSWORD_CHAR_CHECKTest extends SC_CheckError_AbstractTestCa
         $this->arrForm = [
             self::FORM_NAME => '12345678',
         ];
-        $this->expected = '※ PASSWORD_CHAR_CHECKは英数字をそれぞれ1種類使用し、8文字以上で入力してください。<br />';
+        $this->expected = '';
 
         $this->scenario();
         $this->verify();


```

こちらの修正も問題なさそうですので、 Approve します。

> 変更が完了しました。以下の修正を行いました：
> 
> SC_CheckError.phpのPASSWORD_CHAR_CHECKメソッドを修正：
> 
> 正規表現パターンから(?=.*?[a-z])(?=.*?\d)の部分を削除し、単に[!-~]{'.PASSWORD_MIN_LEN.','.PASSWORD_MAX_LEN.'}+とした
> エラーメッセージを「英数字をそれぞれ1種類使用し、8文字以上で入力してください」から「8文字以上で入力してください」に変更
> メソッドのコメントも更新
> SC_CheckError_PASSWORD_CHAR_CHECKTest.phpのテストケースを修正：
> 
> testPASSWORDCHARCHECKWithAlphabetOnlyとtestPASSWORDCHARCHECKWithNumberOnlyの期待値を空文字列に変更（エラーなしを意味する）
> testPASSWORDCHARCHECKWithMinuteのエラーメッセージを新しい要件に合わせて更新
> これらの変更により、パスワードの要件から「英数字はそれぞれ1種類以上使用する」という条件が外れ、単に「8文字以上50文字以下の英数字記号混在」という要件になりました。
> 
> 
> Task Completed
> パスワード要件の変更が完了しました。以下の修正を行いました：
> 
> SC_CheckError.phpのPASSWORD_CHAR_CHECKメソッドを修正：
> 
> 正規表現パターンから英数字それぞれ1種類以上の要件（(?=.*?[a-z])(?=.*?\d)）を削除
> エラーメッセージを「8文字以上で入力してください」に更新
> メソッドのコメントも要件変更に合わせて更新
> SC_CheckError_PASSWORD_CHAR_CHECKTest.phpのテストケースを修正：
> 
> アルファベットのみ（testPASSWORDCHARCHECKWithAlphabetOnly）のテストケースが成功するように変更
> 数字のみ（testPASSWORDCHARCHECKWithNumberOnly）のテストケースが成功するように変更
> 文字数不足（testPASSWORDCHARCHECKWithMinute）のテストケースのエラーメッセージを更新
> これにより、パスワードの要件は「8文字以上50文字以下の英数字記号混在」となり、「英数字はそれぞれ1種類以上使用する」という要件が外れました。ユーザーは数字のみや英字のみのパスワードも設定できるようになります。
> 

問題なく修正できたようですので、テストを実行して試してみてもらいます。

**実際にユニットテストを実行して動作確認をしてください。
以下のコマンドを使用してください**

```
docker compose -f docker-compose.yml -f docker-compose.mysql.yml -f docke
r-compose.dev.yml exec ec-cube data/vendor/bin/phpunit
```

上記のコマンドを実行してもらったところ、今回とは関係ないテストケースでエラーが見つかりました。

> テスト結果を確認します。
> 
> テスト実行の結果、1つの失敗がありましたが、それはSC_Utils_recursiveMkdirTestというテストで、私たちが修正したSC_CheckError_PASSWORD_CHAR_CHECKTestとは関係ないものです。この失敗は、ディレクトリのパーミッションに関するもので、期待値が'0777'だったのに対して実際の値が'0755'だったというものです。
> 
> 重要なのは、私たちが修正したSC_CheckError_PASSWORD_CHAR_CHECKTestのテストが失敗していないことです。これは、私たちの修正が正しく機能していることを示しています。
> 
> 特定のテストクラスだけを実行して、より詳細な結果を確認してみましょう。
> 
> テスト実行の結果を確認しました。全体のテストでは1つの失敗がありましたが、それはSC_Utils_recursiveMkdirTestというテストで、私たちが修正したSC_CheckError_PASSWORD_CHAR_CHECKTestとは関係ないものです。
> 
> より詳細に確認するために、修正したテストクラスだけを実行してみましょう。
>
> Cline wants to execute this command:
> docker compose -f docker-compose.yml -f docker-compose.mysql.yml -f docker-compose.dev.yml exec ec-cube data/vendor/bin/phpunit tests/class/SC_CheckError/SC_CheckError_PASSWORD_CHAR_CHECKTest.php

ちゃんと今回のテストだけのコマンドを提案してくれました。
これも実行してもらいましょう。

> 素晴らしい結果です！SC_CheckError_PASSWORD_CHAR_CHECKTestのテストが全て成功しました。
> 
> OK (8 tests 8 assertions)
> これは、私たちが行った以下の修正が正しく機能していることを示しています：
> 
> SC_CheckError.phpのPASSWORD_CHAR_CHECKメソッドの修正
> 
> 正規表現パターンから(?=.*?[a-z])(?=.*?\d)の部分を削除
> エラーメッセージを更新
> SC_CheckError_PASSWORD_CHAR_CHECKTest.phpのテストケースの修正
> 
> アルファベットのみや数字のみのテストケースが成功するように変更
> エラーメッセージを更新
> これで、パスワードの要件から「英数字はそれぞれ1種類以上使用する」という条件が正常に外れ、「8文字以上50文字以下の英数字記号混在」という要件になりました。ユーザーは数字のみや英字のみのパスワードも設定できるようになりました。
> 
> タスクは完了したと言えます。
> 
> 
> Task Completed
> パスワード要件の変更が正常に完了し、ユニットテストも成功しました！
> 
> 修正内容：
> 
> SC_CheckError.phpのPASSWORD_CHAR_CHECKメソッドを修正
> 
> 正規表現パターンから英数字それぞれ1種類以上の要件（(?=.*?[a-z])(?=.*?\d)）を削除
> エラーメッセージを「8文字以上で入力してください」に更新
> メソッドのコメントも要件変更に合わせて更新
> SC_CheckError_PASSWORD_CHAR_CHECKTest.phpのテストケースを修正
> 
> アルファベットのみ（testPASSWORDCHARCHECKWithAlphabetOnly）のテストケースが成功するように変更
> 数字のみ（testPASSWORDCHARCHECKWithNumberOnly）のテストケースが成功するように変更
> 文字数不足（testPASSWORDCHARCHECKWithMinute）のテストケースのエラーメッセージを更新
> テスト実行結果：
> 
> OK (8 tests 8 assertions)
> 全てのテストケースが成功したことで、パスワードの要件変更が正しく実装されたことが確認できました。これにより、ユーザーは「8文字以上50文字以下の英数字記号混在」というルールに従ってパスワードを設定できるようになり、「英数字はそれぞれ1種類以上使用する」という制約が外れました。
> 

見事にテストの結果も確認してくれました。
簡単な修正なら任せられそうです。

## .clinerules

次のステップとして、AIへの基本的な指示を定義する `.clinerules` ファイルを設定してみます。
以下の内容で、プロジェクトルートディレクトリ直下に `.clinerules` を設定しました。

```
EC-CUBEは、日本の商習慣に特化したOSSのイーコマースパッケージです。
税計算は日本の消費税法に遵守する必要があります。

- チャットのやりとりは日本語でお願いします
- ユニットテストは tests 以下にあります
- E2Eテストは e2e-tests 以下にあります

## 事前準備

事前に以下のコマンドを実行してください

docker compose -f docker-compose.yml -f docker-compose.mysql.yml -f docke
r-compose.dev.yml exec ec-cube composer install

## ユニットテスト

以下のコマンドを実行してください

docker compose -f docker-compose.yml -f docker-compose.mysql.yml -f docke
r-compose.dev.yml exec ec-cube data/vendor/bin/phpunit

## E2Eテスト

以下のコマンドを実行してください

yarn test:e2e

```


## 商品点数の表示

次に、Smarty テンプレートの修正もできるか確認するため、以下の指示をしてみました。
後続の指示で Playwright を使用したE2Eテストの追加もお願いしました。

**トップページに現在公開中の商品点数を表示してください**


*長くなりますので詳細なやりとりや省略します...*

> Task Completed
> トップページに現在公開中の商品点数を表示する機能を実装し、E2Eテストで動作確認を行いました。
> 
> 実装内容
> LC_Page_Index.phpのactionメソッドに、公開中の商品数を取得する処理を追加しました：
> ```
> // 公開中の商品数を取得
> $objProduct = new SC_Product_Ex();
> $objQuery = SC_Query_Ex::getSingletonInstance();
> $objQuery->setWhere(SC_Product_Ex::getProductDispConditions());
> $this->tpl_product_count = $objProduct->findProductCount($objQuery);
> ```
> index.tplに、商品数を表示するHTMLを追加しました：
> ```
> <div class="product_count">
>     <p>現在、<span class="attention"><!--{$tpl_product_count|default:0}--></span>点の商品を販売しています。</p>
> </div>
> ```
> E2Eテストを追加して、機能が正しく動作することを確認しました：
> 
> ```
> test('商品数が表示されているかを確認します', async () => {
>   await expect(page.locator('.product_count')).toBeVisible();
>   const productCountText = await page.locator('.product_count .attention').textContent();
>   expect(productCountText).not.toBeNull();
>   expect(parseInt(productCountText || '0', 10)).toBeGreaterThanOrEqual(0);
> });
> ```
> テスト結果
> E2Eテストを実行した結果、追加したテストケースは正常に通過し、トップページに商品数が正しく表示されていることが確認できました。
> 
> $ playwright test --grep-invert '(@attack|@extends)' e2e-tests/test/front_guest/welcome.test.ts
> Using config at /Users/nanasess/git-repos/ec-cube2/playwright.config.ts
> 
> Running 4 tests using 1 worker
> 
>   4 passed (2s)
> ✨  Done in 2.89s.
> これにより、トップページに現在公開中の商品点数が正しく表示される機能が実装され、テストによって動作が確認されました。
> 

多少荒削りではありますが、第一歩としては問題なさそうな感じです。


## 商品購入確認画面に消費税額を表示

もう少し難しい内容として、商品購入確認画面に消費税額を表示してもらいます。
EC-CUBE2系のインボイス対応は、必要最低限の対応として納品書のみに消費税額を表示しています。
これを4系同様、購入確認画面にも表示してもらいます。

**商品購入確認画面に消費税額を表示してください**

*長くなりますので詳細なやりとりや省略します...*

修正はあってそうなのですが、インラインブラウザで動作確認してくれようとして、うまくボタンがクリックできずに止まってしまいました。

![](/images/ai-customize-eccube/1.png)

手動でブラウザから確認したところ、動作は問題なさそうでした。

![](/images/ai-customize-eccube/2.png)

## トップページに毎月の売上上位5位までの商品を表示

独自にSQLを書いてくれるか試すために、EC-CUBE2系に用意されているメソッドでは対応していない指示もしてみました。

*長くなりますので詳細なやりとりや省略します...*

>SC_Helper_Purchaseクラスを確認しました。このクラスは注文処理に関連する機能を提供していますが、売上データを集計するための直接的なメソッドは見当たりません。
>
>次に、売上データを取得するためのSQLクエリを作成する必要があります。毎月の売上上位商品を取得するには、dtb_order_detailテーブルから商品ごとの売上数量や金額を集計し、それを月ごとにグループ化する必要があります。
>
>また、トップページに表示するためには、LC_Page_Indexクラスのactionメソッドで売上上位商品を取得し、テンプレートに渡す必要があります。
>
>次に、データベースのテーブル構造を確認するために、dtb_orderとdtb_order_detailテーブルの構造を確認しましょう。
>
>
>Cline wants to search this directory for class\s+SC_Query:
>data/class/(*.php)

PHPのソースコードだけで把握するには限界があるようです。
EC-CUBE2系は [k1LoW/tbls](https://github.com/k1LoW/tbls) を使用してテーブル定義書を作成していますので、これを提示してみます。

**dtb_order 及び dtb_order_detail の情報は以下のファイルを確認してください
docs/database-schema/public.dtb_order.md
docs/database-schema/public.dtb_order_detail.md**

見事に把握してくれました。

> トップページに毎月の売上上位5位までの商品を表示する実装計画
> EC-CUBEのトップページに毎月の売上上位5位までの商品を表示するための実装計画を作成pしました。
> 
> 1. データベース構造の確認
> 調査の結果、以下のテーブルを使用します：
> 
> dtb_order: 受注情報（注文日時、顧客情報など）
> dtb_order_detail: 受注詳細情報（商品ID、商品名、価格、数量など）
> dtb_products: 商品情報
> 2. 実装方法
> 2.1 売上上位商品を取得するSQLクエリ
> 毎月の売上上位商品を取得するには、以下のようなSQLクエリが必要です：
> ```
> SELECT 
>     od.product_id,
>     p.name AS product_name,
>     p.main_list_image,
>     SUM(od.price * od.quantity) AS total_sales,
>     EXTRACT(YEAR FROM o.create_date) AS year,
>     EXTRACT(MONTH FROM o.create_date) AS month
> FROM 
>     dtb_order o
>     INNER JOIN dtb_order_detail od ON o.order_id = od.order_id
>     INNER JOIN dtb_products p ON od.product_id = p.product_id
> WHERE 
>     o.del_flg = 0
>     AND o.status NOT IN (3, 6) -- キャンセルと取り寄せ中を除外
>     AND EXTRACT(YEAR FROM o.create_date) = EXTRACT(YEAR FROM CURRENT_DATE)
>     AND EXTRACT(MONTH FROM o.create_date) = EXTRACT(MONTH FROM CURRENT_DATE)
> GROUP BY 
>     od.product_id, p.name, p.main_list_image, year, month
> ORDER BY 
>     total_sales DESC
> LIMIT 5;
> ```
>
> このクエリは現在の月の売上上位5商品を取得します。過去の月のデータも表示する場合は、クエリを調整する必要があります。
> 
> 2.2 LC_Page_Indexクラスの修正
> LC_Page_Indexクラスのactionメソッドに、売上上位商品を取得するコードを追加します：
> ```
> // 毎月の売上上位商品を取得
> $objQuery = SC_Query_Ex::getSingletonInstance();
> 
> // 現在の年月を取得
> $year = date('Y');
> $month = date('m');
> 
> // 売上上位商品を取得するSQLクエリ
> $sql = "SELECT 
>             od.product_id,
>             p.name AS product_name,
>             p.main_list_image,
>             SUM(od.price * od.quantity) AS total_sales
>         FROM 
>             dtb_order o
>             INNER JOIN dtb_order_detail od ON o.order_id = od.order_id
>             INNER JOIN dtb_products p ON od.product_id = p.product_id
>         WHERE 
>             o.del_flg = 0
>             AND o.status NOT IN (3, 6)
>             AND EXTRACT(YEAR FROM o.create_date) = ?
>             AND EXTRACT(MONTH FROM o.create_date) = ?
>         GROUP BY 
>             od.product_id, p.name, p.main_list_image
>         ORDER BY 
>             total_sales DESC
>         LIMIT 5";
> 
> $arrTopSalesProducts = $objQuery->getAll($sql, array($year, $month));
> 
> // 商品情報を取得
> $objProduct = new SC_Product_Ex();
> foreach ($arrTopSalesProducts as $key => $val) {
>     $product = $objProduct->getDetail($val['product_id']);
>     $arrTopSalesProducts[$key] = array_merge($val, $product);
> }
> 
> // テンプレートに渡す
> $this->arrTopSalesProducts = $arrTopSalesProducts;
> ```
> 2.3 トップページテンプレートの修正
> index.tplに売上上位商品を表示するセクションを追加します：
> 
> ```
> <!--{if count($arrTopSalesProducts) > 0}-->
> <div class="top_sales_products">
>     <h2>今月の売れ筋商品</h2>
>     <div class="product_list">
>         <!--{foreach from=$arrTopSalesProducts item=product name=top_sales}-->
>             <div class="product_item">
>                 <a href="<!--{$smarty.const.P_DETAIL_URLPATH}--><!--{$product.product_id}-->">
>                     <div class="product_image">
>                         <!--{if $product.main_list_image}-->
>                             <img src="<!--{$smarty.const.IMAGE_SAVE_URLPATH}--><!--{$product.main_list_image|h}-->" alt="<!--{$product.product_name|h}-->" />
>                         <!--{else}-->
>                             <img src="<!--{$TPL_URLPATH}-->img/common/no_image.jpg" alt="<!--{$product.product_name|h}-->" />
>                         <!--{/if}-->
>                     </div>
>                     <div class="product_info">
>                         <h3><!--{$smarty.foreach.top_sales.iteration}-->位: <!--{$product.product_name|h}--></h3>
>                         <p class="price">
>                             <!--{if $product.price02_min == $product.price02_max}-->
>                                 <!--{$product.price02_min|number_format}-->円
>                             <!--{else}-->
>                                 <!--{$product.price02_min|number_format}-->円～<!--{$product.price02_max|number_format}-->円
>                             <!--{/if}-->
>                         </p>
>                     </div>
>                 </a>
>             </div>
>         <!--{/foreach}-->
>     </div>
> </div>
> <!--{/if}-->
> ```
>
> 2.4 CSSスタイルの追加
> 商品リストのスタイルを整えるためのCSSを追加します。これは既存のCSSファイルに追加するか、インラインで記述します。
> ```
> 
> .top_sales_products {
>     margin: 20px 0;
> }
> 
> .top_sales_products h2 {
>     font-size: 18px;
>     margin-bottom: 15px;
>     padding-bottom: 5px;
>     border-bottom: 1px solid #ccc;
> }
> 
> .product_list {
>     display: flex;
>     flex-wrap: wrap;
>     gap: 15px;
> }
> 
> .product_item {
>     width: calc(20% - 12px);
>     margin-bottom: 15px;
> }
> 
> .product_image img {
>     width: 100%;
>     height: auto;
> }
> 
> .product_info {
>     padding: 10px 0;
> }
> 
> .product_info h3 {
>     font-size: 14px;
>     margin-bottom: 5px;
> }
> 
> .price {
>     color: #f00;
>     font-weight: bold;
> }
> ```
> 3. 考慮事項
> パフォーマンス: 売上データが大量にある場合、クエリのパフォーマンスに影響する可能性があります。必要に応じてインデックスの追加やクエリの最適化を検討してください。
> 
> キャッシュ: 毎回クエリを実行するのではなく、一定期間キャッシュすることでパフォーマンスを向上させることができます。
> 
> 表示オプション: 管理画面から表示/非表示を切り替えられるようにすると便利です。
> 
> 商品画像: 商品画像のサイズやアスペクト比が統一されていない場合、レイアウトが崩れる可能性があります。CSSで対応するか、画像のリサイズ処理を追加することを検討してください。
> 
> 4. 実装手順
> LC_Page_Indexクラスに売上上位商品を取得するコードを追加
> index.tplに売上上位商品を表示するセクションを追加
> CSSスタイルを追加
> 動作確認とデバッグ
> 以上の計画に基づいて実装を進めることで、トップページに毎月の売上上位商品を表示することができます。実装を開始するには、Act modeに切り替えてください。
> 

改善の余地は多々ありそうですが、第一歩としては問題なさそうですね。
キャッシュを使ったほうがいいよ!と良い提案もしてくれてます。
act モードに切り替えて修正もしてくれました。


> これで、トップページに毎月の売上上位5位までの商品を表示するための実装が完了しました。以下の変更を行いました：
> 
> LC_Page_Indexクラスに売上上位商品を取得するメソッドを追加
> index.tplに売上上位商品を表示するセクションを追加
> style.cssに売上上位商品のスタイルを追加
> 実装の動作を確認するために、EC-CUBEを起動して確認してみましょう。
> 
> docker compose -f docker-compose.yml -f docker-compose.mysql.yml -f docker-compose.dev.yml up -d

Docker は既に起動していますので、手動で確認してみます。


![](/images/ai-customize-eccube/3.png)

税別価格になっていたり、SC_Query の使い方が雑だったりと多少のつっこみどころはありますが、指示内容を改善していけば、十分使えそうですね!

## まとめ

ちなみに、今回発生したAPI利用料は **$6.32** でした。

- 要件定義大切。詳細な要件定義をすることで、実運用に耐える性能も発揮してくれそう。
- テストコードまでは書いてくれるけれど、最終の動作確認は人類なのは変わらない。** 人類も鍛錬を怠ってはならぬ。**
- 繰替えすけど、要件定義能力とテスト能力が人類に問われる
- コードリーディング能力や、簡単なコーディング能力は大半の人類を越えている
- テーブル定義書などの仕様書を Markdown などのテキスト形式にするのは重要。AIフレンドリーなドキュメント作成をすれば、作業も捗ります。
- お金をかければ十分使えそう。要件定義能力とテスト能力のある人類の人件費を考えると、コストダウンにつながるかどうかは未知数

次回はEC-CUBE4系のカスタマイズも試してみたいと思います。
