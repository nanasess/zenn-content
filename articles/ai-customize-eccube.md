---
title: "AIに命令してどこまでEC-CUBEをカスタマイズできるか? | EC-CUBE名古屋 vol.113"
emoji: "🙆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["eccube","Cline"]
published: false
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


## .clinerules

```
EC-CUBEは、日本の商習慣に特化したOSSのイーコマースパッケージです。
税計算は日本の消費税法に遵守する必要があります。

- チャットのやりとりは日本語でお願いします
- ユニットテストは tests 以下にあります
- E2Eテストは e2e-tests 以下にあります

## 事前準備

事前に以下のコマンドを実行してください

```
docker compose -f docker-compose.yml -f docker-compose.mysql.yml -f docke
r-compose.dev.yml exec ec-cube composer install
```

## ユニットテスト

以下のコマンドを実行してください

```
docker compose -f docker-compose.yml -f docker-compose.mysql.yml -f docke
r-compose.dev.yml exec ec-cube data/vendor/bin/phpunit
```

## E2Eテスト

以下のコマンドを実行してください

```
yarn test:e2e
```

```

```

## 商品点数の表示

トップページに現在公開中の商品点数を表示してください

## 商品購入確認画面に消費税額を表示してください
