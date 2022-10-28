---
title: "10分でEC-CUBEプラグインのE2Eテストを書いてみる"
emoji: "⏱️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ECCUBE", "Plugin"]
published: true
---
EC-CUBEプラグイン開発者のみなさん、E2Eテストは作ってますか？
GitHub Actions でテストの自動化はしていますか？

従来、EC-CUBEのプラグインのE2Eテストをするためには、さまざまなステップが必要でした。
多くの方は、手動でテストされていると思います。

別記事である**Composerを使いこなしてEC-CUBE4系プラグインの開発効率を爆上げする**を応用して、10分で簡単にE2Eテストを書いてみましょう！

https://zenn.dev/nanasess/articles/ec-cube4-plugin-development

**この方法は、EC-CUBE4.0.x〜4.2.x まで使用可能です。**

2系のプラグイン/決済モジュールについても応用可能ですので、順次まとめたいと思います。

## 前提条件

- [EC-CUBE4.0.xのみ] EC-CUBEオーナーズストアの認証キーを取得しておきます
- Node.js 14 以上がインストールされていること
- docker-compose が利用できること

Windows の方は、 WSL2 の利用をおすすめします。

## プラグインを用意

まず、開発したいプラグインを用意します。
ここでは [SimplePayment](https://github.com/EC-CUBE/sample-payment-plugin) プラグインを例にします。
プラグイン名や、プラグインコードなどは適宜読み替えてください。

```shell
git clone https://github.com/EC-CUBE/sample-payment-plugin
```

プラグインを一から開発したい方は、公式ドキュメント [プラグインを開発する#プラグインの雛形を生成](https://doc4.ec-cube.net/plugin_development#%E3%83%97%E3%83%A9%E3%82%B0%E3%82%A4%E3%83%B3%E3%81%AE%E9%9B%9B%E5%BD%A2%E3%82%92%E7%94%9F%E6%88%90) を参考に雛形を生成し、 `app/Plugin/<プラグインコード>` ディレクトリを EC-CUBE とは別の階層にコピーしておいてください。

用意できたら、プラグインのディレクトリに移動します。

```shell
cd sample-payment-plugin
```
## docker-compose.yml と docker-compose.dev.yml を用意

以下をコピーして、 docker-compose.yml と docker-compose.dev.yml を用意します。

### EC-CUBE4.0.x の場合

docker-compose.dev.yml にオーナーズストアの認証キーと、プラグインコードを設定しておきます。

```yaml
## docker-compose.yml
version: "3"

networks:
  backend:
    driver: bridge

volumes:
  mailcatcher-data:
    driver: local

  ### ignore folder volume #####
  var:
    driver: local
  vendor:
    driver: local
  node_modules:
    driver: local

services:
  ### ECCube4 ##################################
  ec-cube:
    image: ghcr.io/nanasess/ec-cube-php:7.4-apache-4.0
    ports:
      - 8080:80
      - 4430:443
    volumes:
      - "var:/var/www/html/var"
      - "vendor:/var/www/html/vendor"
      - "node_modules:/var/www/html/node_modules"
    environment:
      # EC-CUBE environments
      APP_ENV: "dev"
      APP_DEBUG: 1
      DATABASE_URL: "sqlite:///var/eccube.db"
      DATABASE_SERVER_VERSION: 3
      DATABASE_CHARSET: 'utf8'
      MAILER_DSN: "smtp://mailcatcher:1025"
      ECCUBE_AUTH_MAGIC: "<change.me>"
    networks:
      - backend

  ### Mailcatcher ##################################
  mailcatcher:
    image: schickling/mailcatcher
    ports:
      - "1080:1080"
      - "1025:1025"
    networks:
      - backend
```

```yaml
## docker-compose.dev.yml
version: '3'

services:
  ec-cube:
    entrypoint: >
      /bin/bash -c "
      docker-php-entrypoint ls &&
      bin/console doctrine:query:sql \"UPDATE dtb_base_info SET authentication_key = '$${ECCUBE_AUTHENTICATION_KEY}'\" &&
      composer config repositories.plugin '{\"type\": \"path\", \"url\": \"../plugin\"}' &&
      bin/console eccube:composer:require ec-cube/$${PLUGIN_CODE} &&
      bin/console eccube:plugin:enable --code=$${PLUGIN_CODE} &&
      apache2-foreground
      "
    environment:
      PLUGIN_CODE: <プラグインコード>
      ECCUBE_AUTHENTICATION_KEY: <オーナーズストアの認証キー>
    volumes:
      - ".:/var/www/plugin:cached"
```

### EC-CUBE4.1.x の場合

docker-compose.dev.yml にプラグインコードとプラグイン名を設定しておきます。

```yaml
## docker-compose.yml
version: "3"

networks:
  backend:
    driver: bridge

volumes:
  mailcatcher-data:
    driver: local

  ### ignore folder volume #####
  var:
    driver: local
  vendor:
    driver: local
  node_modules:
    driver: local

services:
  ### ECCube4 ##################################
  ec-cube:
    image: ghcr.io/nanasess/ec-cube-php:7.4-apache-4.1
    ports:
      - 8080:80
      - 4430:443
    volumes:
      - "var:/var/www/html/var"
      - "vendor:/var/www/html/vendor"
      - "node_modules:/var/www/html/node_modules"
    environment:
      # EC-CUBE environments
      APP_ENV: "dev"
      APP_DEBUG: 1
      DATABASE_URL: "sqlite:///var/eccube.db"
      DATABASE_SERVER_VERSION: 3
      DATABASE_CHARSET: 'utf8'
      MAILER_DSN: "smtp://mailcatcher:1025"
      ECCUBE_AUTH_MAGIC: "<change.me>"
    networks:
      - backend

  ### Mailcatcher ##################################
  mailcatcher:
    image: schickling/mailcatcher
    ports:
      - "1080:1080"
      - "1025:1025"
    networks:
      - backend
```

```yaml
## docker-compose.dev.yml
version: '3'

services:
  ec-cube:
    entrypoint: >
      /bin/bash -c "
      docker-php-entrypoint ls &&
      composer config repositories.plugin '{\"type\": \"path\", \"url\": \"../plugin\"}' &&
      bin/console eccube:composer:require $${PLUGIN_NAME} &&
      bin/console eccube:plugin:enable --code=$${PLUGIN_CODE} &&
      apache2-foreground
      "
    environment:
      PLUGIN_CODE: <プラグインコード> # 例) SamplePayment
      PLUGIN_NAME: ec-cube/<すべて小文字のPluginCode> # 例) ec-cube/samplepayment4
    volumes:
      - ".:/var/www/plugin:cached"

```

### EC-CUBE4.2.x の場合

docker-compose.dev.yml にプラグインコードとプラグイン名を設定しておきます。

```yaml
## docker-compose.yml
version: "3"

networks:
  backend:
    driver: bridge

volumes:
  mailcatcher-data:
    driver: local

  ### ignore folder volume #####
  var:
    driver: local
  vendor:
    driver: local
  node_modules:
    driver: local

services:
  ### ECCube4 ##################################
  ec-cube:
    image: ghcr.io/ec-cube/ec-cube-php:7.4-apache-4.2
    ports:
      - 8080:80
      - 4430:443
    volumes:
      - "var:/var/www/html/var"
      - "vendor:/var/www/html/vendor"
      - "node_modules:/var/www/html/node_modules"
    environment:
      # EC-CUBE environments
      APP_ENV: "dev"
      APP_DEBUG: 1
      DATABASE_URL: "sqlite:///var/eccube.db"
      DATABASE_SERVER_VERSION: 3
      DATABASE_CHARSET: 'utf8'
      MAILER_DSN: "smtp://mailcatcher:1025"
      ECCUBE_AUTH_MAGIC: "<change.me>"
    networks:
      - backend

  ### Mailcatcher ##################################
  mailcatcher:
    image: schickling/mailcatcher
    ports:
      - "1080:1080"
      - "1025:1025"
    networks:
      - backend
```

```yaml
## docker-compose.dev.yml
version: '3'

services:
  ec-cube:
    entrypoint: >
      /bin/bash -c "
      docker-php-entrypoint ls &&
      composer config repositories.plugin '{\"type\": \"path\", \"url\": \"../plugin\"}' &&
      bin/console eccube:composer:require $${PLUGIN_NAME} &&
      bin/console eccube:plugin:enable --code=$${PLUGIN_CODE} &&
      apache2-foreground
      "
    environment:
      PLUGIN_CODE: <プラグインコード> # 例) SamplePayment
      PLUGIN_NAME: ec-cube/<すべて小文字のPluginCode> # 例) ec-cube/samplepayment4
    volumes:
      - ".:/var/www/plugin:cached"

```

## EC-CUBE を起動する

以下のコマンド一発で、プラグインインストール済みでセットアップされたEC-CUBEが起動します。

```shell
docker-compose up -d -f docker-compose.yml -f docker-compose.dev.yml
```

http://localhost:8080 へアクセスしてフロント画面が起動しているのを確認してください。
管理画面は http://localhost:8080/admin/ です。(ID: admin, PASS: password)

### (オプション) 起動時にプラグインの初期設定を完了する

docker-compose.dev.yml の `entrypoint` にスクリプトを書いておくことで、EC-CUBEの起動時にプラグインの初期設定が完了した状態にできます。
これを活用することで、E2Eテストの自動化が格段にしやすくなります。

```yaml
### docker-compose.dev.yml の entrypoint の例です
### dtb_payment_option に支払い方法の設定を INSERT しておくことで、
### EC-CUBE 起動直後にプラグインを利用可能としています。
    entrypoint: >
      /bin/bash -c "
      docker-php-entrypoint ls &&
      bin/console doctrine:query:sql \"UPDATE dtb_base_info SET authentication_key = '$${ECCUBE_AUTHENTICATION_KEY}'\" &&
      composer config repositories.plugin '{\"type\": \"path\", \"url\": \"../plugin\"}' &&
      bin/console eccube:composer:require ec-cube/SamplePayment &&
      bin/console eccube:plugin:enable --code=SamplePayment &&
      bin/console doctrine:query:sql \"INSERT INTO dtb_payment_option VALUES(1,7,'paymentoption');\" &&
      bin/console doctrine:query:sql \"INSERT INTO dtb_payment_option VALUES(1,6,'paymentoption');\" &&
      bin/console doctrine:query:sql \"INSERT INTO dtb_payment_option VALUES(1,5,'paymentoption');\" &&
      apache2-foreground
      "
```

## playwright のセットアップ

E2Eテストを実行するための Playwright をセットアップします。

```
npm init playwright@latest
```

上記のコマンドを実行するといろいろ聞かれますが、特にこだわりなければ yes で良いと思います。
*Do you want to use TypeScript or JavaScript?* は TypeScript がおすすめです。

## テストコードを自動生成する

Playwright には、E2Eテストのコードを自動生成する機能があります。
これを使って、E2Eテストを書いてみましょう！

```
npx playwright codegen http://localhost:8080
```

上記のコマンドを実行すると、ブラウザと **Playwright Inspector** が自動で起動します。
EC-CUBEのトップページが表示されていると思いますので、プラグインの機能を操作してみてください。
**Playwright Inspector** にテストコードが自動生成されていきます。

## テストコードをコピーして実行してみる

Playwright をセットアップすると、`tests` というディレクトリが生成されます。
この中に `example.spec.ts` というテストファイルが生成されています。

このファイルに、先ほど生成されたテストコードを貼りつけましょう。

以下の例は、商品をカートに投入するテストコードです。

```typescript
import { test, expect } from '@playwright/test';

test('test', async ({ page }) => {

  await page.goto('http://localhost:8080/');

  await page.getByRole('link', { name: '新入荷' }).click();
  await expect(page).toHaveURL('http://localhost:8080/products/list?category_id=2');

  await page.getByRole('link', { name: 'チェリーアイスサンド ￥3,080' }).click();
  await expect(page).toHaveURL('http://localhost:8080/products/detail/2');

  await page.getByRole('button', { name: 'カートに入れる' }).click();

  await page.getByRole('link', { name: 'カートへ進む' }).click();
  await expect(page).toHaveURL('http://localhost:8080/cart');

});
```

## テストを実行

以下のコマンドでテストを実行してみましょう！

```
npx playwright test
```

デフォルトでは、 Chromium(Chrome), Firefox, Webkit(Safari) が自動的に起動してテストが実行されます。

起動するブラウザを変更したい場合は、 `playwright.config.ts` の `projects` の項目を編集してください。

## (おまけ) GitHub Actions で実行する

`.github/workflows/playwright.yml` が生成されていると思います。
このファイルの `- name: Run Playwright tests` の前あたりに以下のコードを記述してください。

```yaml
    - name: Setup EC-CUBE
      env:
        PLUGIN_CODE: <プラグインコード>
        PLUGIN_NAME: ec-cube/<すべて小文字のプラグインコード>
        # EC-CUBE 4.0.x のみ
        ECCUBE_AUTHENTICATION_KEY: ${{ secrets.ECCUBE_AUTHENTICATION_KEY }}
      run: docker compose up -d
```

EC-CUBE 4.0.x の場合は、オーナーズストアの認証キーを [GitHub の secrets](https://docs.github.com/ja/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository) に登録してください。

くれぐれも機密情報をリポジトリに push しないようご注意ください。

## まとめ

Composer と docker-compose を使いこなせば、プラグインの開発効率が爆上がりします。
いままで、**E2Eテストする工数が取れない...** とか、 **環境構築が面倒で...** と思っていた方も、この方法なら10分でできますのでぜひ挑戦してみてください！
