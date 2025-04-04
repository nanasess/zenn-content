---
title: "Codeception で任意のバージョンの Chrome を起動したい"
emoji: "🌏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["eccube","codeception","chrome","php","e2e"]
published: true
---
Chrome や ChromeDriver のバージョンが上がって、Codeception のE2Eテストが突然エラーになったりした時、過去の Chrome や ChromeDriver のバージョンを使いたい場合があります。

そんな時は [@puppeteer/browsers](https://pptr.dev/browsers-api) を使って、 [Chrome for Testing](https://googlechromelabs.github.io/chrome-for-testing/) から過去のバージョンの Chrome をダウンロードしましょう。

## 設定方法

プロジェクトのディレクトリで以下のコマンドを実行します

### [@puppeteer/browsers](https://pptr.dev/browsers-api) をインストール

``` shell
npm install @puppeteer/browsers
```

### 任意のバージョンの Chrome と ChromeDriver をインストール

``` shell
## 安定板の chrome をインストール
npx @puppeteer/browsers install chrome@stable

## バージョン指定で chrome をインストール
npx @puppeteer/browsers install chrome@116.0.5793.0

## Canary チャンネル(開発版)の ChromeDriver をインストール
npx @puppeteer/browsers install chromedriver@canary

# バージョン指定で ChromeDriver をインストール
npx @puppeteer/browsers install chromedriver@116.0.5793.0
```

インストールした際に、インストール先のパスが表示されるのでメモしておいてください

### acceptance.suite.yml ファイルに設定

Codeception の acceptance.suite.yml ファイルに設定します。

``` yaml
actor: AcceptanceTester
modules:
    enabled:
        - WebDriver:
            url: 'http://localhost'
            browser: 'chrome'
            capabilities:
                goog:chromeOptions:
                    binary: '<インストールしたChromeのパス>'
                    prefs:
                        download.default_directory: '%PWD%/codeception/_support/_downloads'
                    args: ['--headless', '--disable-gpu']
```

## 実行方法

インストールした ChromeDriver を起動します。 `--verbose` をつけると通信内容を出力しますのでお好みで。

``` shell
./<インストールしたChromeDriverのパス>/chromedriver --url-base=/wd/hub --port=9515 --verbose
```

あとは通常通り codeception を実行します

``` shell
vendor/bin/codecept  run -d acceptance
```
