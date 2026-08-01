---
title: "【初心者向け】Astro で EC-CUBE のコンテンツ管理を安全にしよう | EC-CUBE名古屋 vol.124"
emoji: "🛡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["eccube", "astro", "wordpress", "cloudflare", "claudecode"]
published: false
---

*この記事は「EC-CUBE名古屋 vol.124」勉強会の記録です。*

当日のスライドはこちらで公開しています。

https://claude.ai/code/artifact/de61b1b5-7660-4ca2-8435-19aa5de90407

:::message
本記事のドメイン名・サーバー名・パス・IPアドレスはすべてダミーに置き換えています（`example.shop` / `ssh.example.jp` など）。実際のデモは筆者管理下の環境で実施しました。
:::

## この記事でやること

EC-CUBE を運営していると、SEO やコンテンツマーケティングのために **WordPress を同居させる**ケースが非常に多くあります。EC-CUBE の CMS 機能（デザイン管理・コンテンツ管理）だけでは記事を量産する用途に向かないためです。

判断としては合理的です。EC-CUBE はそもそもカートシステムであり、単純なページ生成は他に任せたほうが理にかなっています。

しかし現実には、**WordPress を経由した侵入から EC-CUBE 側の個人情報・カード情報が流出する**事故が繰り返し起きています。

そこで今回の提案です。

> **ブログを WordPress から Astro（静的サイトジェネレータ）に置き換えてしまえば、そもそも乗っ取る対象が消えるのではないか。**

勉強会では、これを**実際に本番相当の環境で最後までやりきりました**。この記事はその記録です。

![Astro で作り直したブログの記事一覧。WordPress のデザインをそのまま踏襲している](/images/wordpress-to-astro-vol124/astro-blog-list.png)
*最終的にできあがったブログ。WordPress から10本の記事を移行し、URL もデザインもそのまま引き継いでいる*

---

## ⚠️ 実施にあたっての注意点

**この記事の内容をそのまま本番環境で実行しないでください。** 勉強会では以下の前提を満たしたうえで実施しています。

### 1. クレデンシャルは 1Password から実行時にだけ渡す

Cloudflare の API トークンは、**ファイルにもシェル履歴にも平文で残さない**運用にしています。

```bash
# 1Password CLI から実行時にだけ解決する
TOKEN=$(op read 'op://<vault>/cloudflare-credentials/CLOUDFLARE_API_TOKEN')
curl -H "Authorization: Bearer $TOKEN" https://api.cloudflare.com/client/v4/...
```

継続的に使う場合は `op run` でテンプレート経由の注入も使えます。

```bash
op run --env-file=.env.tpl -- npx wrangler pages deploy dist
```

:::message alert
`.env` に生のトークンを書く、`export CLOUDFLARE_API_TOKEN=xxxx` をシェル履歴に残す、といった運用は避けてください。トークンが漏れると DNS を書き換えられます。
:::

また、**トークンの権限は必要最小限に絞ってください**。今回のデモでは当初 DNS の編集権限しか持たせておらず、Workers ルートの作成が必要になった段階で、その権限だけを追加しました。

### 2. サーバー接続は SSH 鍵認証で

サーバー操作はすべて **SSH 公開鍵認証**で行っています。パスワード認証は無効化し、秘密鍵は 1Password の SSH エージェント（`~/.1password/agent.sock`）で管理しています。

```bash
# ~/.ssh/config
Host ssh.example.jp
    User deployer
    IdentityAgent ~/.1password/agent.sock
```

FTP を使う場合も、暗号化される **SFTP / FTPS** が使えないかサーバー会社に確認してください。パスワードが平文で流れる古い FTP は避けたいところです。

### 3. 環境操作はエンジニアの監視下で実施する

今回のデモは、**AIエージェント（Claude Code）に作業させつつ、各コマンドの内容をエンジニアが確認しながら**進めました。

具体的には、

- DNS 変更・ファイル削除・DB 削除など**取り返しのつかない操作は、実行前に計画を提示して承認を得る**
- 削除の前には必ずバックアップを取り、**バックアップの中身を検証してから**削除する
- 破壊的な操作の後は、**関係ない部分が壊れていないかを機械的に検証**する

という運用です。実際、この記事の後半で触れるとおり、**AIの検証が甘くて事故を起こした場面もありました**。エンジニアの目視確認は省略できません。

---

## 前半：なぜ WordPress の同居が危ないのか

### よくある3つの同居パターン

現場で見かける構成は、だいたいこの3つに分類できます。

| | 構成 | 説明 |
| --- | --- | --- |
| **A** | `https://example.shop` ＋ `/blog` | EC-CUBE が本体、ブログを下にぶら下げる |
| **B** | トップが WordPress、下層に EC-CUBE を混在 | 見た目は完全に一体化するが、最も深く絡み合う |
| **C** | `https://example.jp` ＋ `/shop` | メディアが本体、お店を下にぶら下げる |

パターン B は EC-CUBE 2系で長く使われてきた構成ですが、**4系では実現が困難**です。理由は後述しますが、4系は「すべての URL を `index.php` 1つで受け止める」フロントコントローラ方式のため、WordPress と `index.php` を奪い合ってしまいます。

今回のデモはパターン A で構築しました。

### 「プラグインを入れなければ安全」は、もう通用しない

WordPress が狙われる、という話をすると「うちは余計なプラグインを入れていないから」という反応をいただくことがあります。

しかし 2026年7月、**wp2shell** と名付けられた問題が公表されました。

- **CVE-2026-60137** — `WP_Query` の `author__not_in` パラメータの SQL インジェクション
- **CVE-2026-63030** — REST ルートの route confusion による認可バイパス

この2つを組み合わせると、**認証なしでリモートコード実行**が成立します。しかも、

> **プラグインもテーマも一切入れていない、インストールしたままの WordPress で成立しました。**

対象は WordPress 6.8 系・6.9 系・7.0 系。修正版は **6.8.6 / 6.9.5 / 7.0.2** です。危険度から WordPress 側が強制自動更新に踏み切り、米 CISA も「実際に悪用されている脆弱性」として期限付きの対応を求めました。

参考リンク：

- [WP2Shell WordPress Vulnerabilities Exploited in the Wild — SecurityWeek](https://www.securityweek.com/wp2shell-wordpress-vulnerabilities-exploited-in-the-wild/)
- [WP2Shell Vulnerabilities: CVE-2026-60137 and CVE-2026-63030 — VulnCheck](https://www.vulncheck.com/blog/wp2shell)
- [wp2shell: WordPress Core Pre-Auth RCE FAQ — Tenable](https://www.tenable.com/blog/wp2shell-cve-2026-63030-cve-2026-60137-frequently-asked-questions-about-remote-code-execution)

### 同じサーバーに置く、ということの意味

ブログと EC-CUBE が同じサーバーにあれば、ファイルも、多くの場合データベースも同じ場所にあります。**ブログの鍵が破られた時点で、攻撃者はもう家の中にいます。** EC-CUBE のログイン画面を破る必要がありません。

攻撃者が欲しいのはブログの記事ではなく、その隣にある注文情報とカード情報です。

### 実際に観測されたスキャン

デモ環境を公開した**数分後**、Apache のエラーログにこんな記録が残りました。

```
client denied by server configuration: .../.env
client denied by server configuration: .../wp-config.php.bak
client denied by server configuration: .../secrets.yaml
client denied by server configuration: .../.env.vault
```

EC-CUBE の `.htaccess` が全部弾いていますが、**24時間巡回している**というのは比喩ではありません。防御が1枚剥がれれば通ります。

---

## 後半：実演の記録

ここからは、実際にやったことを順に記録します。**うまくいかなかった部分もそのまま書きます。**

### デモ環境

| 項目 | 内容 |
| --- | --- |
| ホスト | Enterprise Linux 8 系 / Apache + PHP-FPM 7.4 |
| DB | MariaDB 10.3 |
| EC-CUBE | **4.2 系**（PHP 7.4 で動作するため） |
| WordPress | 7.0.2（wp2shell 修正版） |
| ブログ | Astro（blog テンプレート） |
| DNS / CDN | Cloudflare（proxy 有効） |

EC-CUBE と WordPress でデータベースを分け、接続ユーザーも別にしています。

### 1. Astro を導入する

AIエージェントへの指示は、これだけです。

```
うちのお店のブログを Astro で作りたいです。
・タイトルは「〇〇 公式ブログ」
・https://example.shop/blog/ で公開します
記事は空でいいので、まず動く状態まで作ってください。
```

`npm create astro@latest -- --template blog` で雛形ができ、`src/content/blog/` に `.md` を置けばそれが記事になります。**ここが WordPress の管理画面にあたる部分**です。

### 2. 記事を書く

```
新しい記事を書いてください。
テーマ：一色産うなぎの白焼きを始めたこと
産地のこだわりと、おすすめの食べ方を入れてください。
堅すぎない、お客様に語りかける感じで。
```

出力されるのは Markdown ファイル1つ（約1.6KB）です。

```markdown
---
title: '一色産うなぎの白焼き、はじめました'
description: '愛知県西尾市一色町から、この夏いちばんの白焼きが届きました。'
pubDate: 'Aug 01 2026'
---

愛知県西尾市一色町。うなぎの産地として知られるこの町から…
```

**コマンドを一度も手打ちしていません。** ここが「初心者向け」の肝です。

### 3. ビルドして、出てくるものを見る

```bash
npm run build
```

3秒ほどで完了し、`dist/` に成果物が出ます。

```
dist/
├── index.html
├── about/index.html
├── blog/index.html
├── blog/unagi-shirayaki/index.html
├── _astro/fonts/...
└── favicon.ico

合計サイズ : 208K
HTMLファイル: 4 個
PHPファイル : 0 個   ← ここ
データベース: 不要
```

**PHP ファイル 0個、データベース不要。** これがこの手法の全てです。乗っ取る対象となるプログラムが存在しません。

### 4. アップロードしたら 404 になった（4系の落とし穴）

`dist/` をサーバーの `/astro-test/` に配置したところ、**全ページが 404** になりました。

```
/astro-test/                      → 404
/astro-test/blog/                 → 404
/astro-test/blog/unagi-shirayaki/ → 404

/astro-test/index.html            → 200  ← ファイル名まで打てば出る
```

原因は EC-CUBE 4系の `.htaccess` です。

```apache
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !^(.*)\.(gif|png|jpe?g|css|ico|js|svg|map)$ [NC]
RewriteRule ^(.*)$ index.php [QSA,L]
```

`!-f`（ファイルでなければ）はありますが、**`!-d`（ディレクトリでなければ）がありません**。Astro のデフォルト出力はディレクトリ形式 URL（`/blog/post/index.html`）なので、`/astro-test/` は「ファイルではない」と判定されて `index.php` に渡り、Symfony が 404 を返します。

修正は1行です。

```diff
 RewriteCond %{REQUEST_FILENAME} !-f
+RewriteCond %{REQUEST_FILENAME} !-d
 RewriteCond %{REQUEST_FILENAME} !^(.*)\.(gif|png|jpe?g|css|ico|js|svg|map)$ [NC]
 RewriteRule ^(.*)$ index.php [QSA,L]
```

:::message
**2系ではこの問題は起きません。** 2系は `html/` に catch-all の rewrite が無いため、ディレクトリを置くだけで素直に表示されます。この差が、冒頭の「パターン B は4系では困難」という話にもつながります。
:::

### 5. テンプレートのサンプルページが残っていた

`.htaccess` を直して表示できるようになったものの、トップページを開くとこうなっていました。

![Astro の blog テンプレート付属のサンプルページ「Hello, Astronaut!」が表示されている](/images/wordpress-to-astro-vol124/astro-sample-page.png)
*テンプレート付属の英語サンプルがそのまま出ている状態*

Astro の blog テンプレートは `src/pages/index.astro` に英語のサンプルを持っています。WordPress でいう「Hello world!」と同じで、**消し忘れるとそのまま公開されます**。

記事一覧は `/blog/` 側にあり、トップページとは別物でした。今回は WordPress と同じ構造（トップ＝一覧）に寄せるため、`index.astro` を記事一覧に書き換えました。

### 6. サブディレクトリ配置でリンクが全部ズレる

Astro を `base: '/blog'` などサブパスに置くと、**テンプレート内のリンクが `base` を無視して `/` を指したまま**になります。今回はヘッダーの「Home」が `/`（＝EC-CUBE のトップ）に飛ぶ状態でした。

```diff
- <HeaderLink href="/">Home</HeaderLink>
+ <HeaderLink href={`${base}/`}>ホーム</HeaderLink>
```

厄介なのは、**1箇所直しても別の場所に残る**ことです。実際、記事一覧ページ（`src/pages/blog/index.astro`）と `BaseHead.astro` の favicon / sitemap の指定にも同じ問題が残っていて、指摘を受けて気づきました。

これは人間の目視では必ず取りこぼします。**AIには「全部探して直せ」と指示するのが有効**です。

```
base が付いていないリンクを全部探して直してください
```

```bash
$ rg 'href=("/|\{`/)' src/ -g '*.astro' | grep -v 'https://'
src/components/Header.astro:14:   <HeaderLink href="/">オンラインショップ</HeaderLink>  ← 意図的
src/components/BaseHead.astro:24: <link rel="icon" href="/favicon.svg" />
src/pages/blog/index.astro:97:   <a href={`/blog/${post.id}/`}>
```

### 7. WordPress の記事を Astro へ移行する

記事10本を wp-cli で書き出し、HTML → Markdown に変換しました。

```bash
wp post list --post_type=post --fields=ID,post_name,post_title,post_date --format=json
wp post get <ID> --field=post_content
```

**ここで一番はまったのが URL の維持です。**

WordPress は日本語スラッグを**URL エンコードした状態で保存**しています。

```
post_name: %e4%b8%80%e8%89%b2%e7%94%a3%e3%81%86%e3%81%aa%e3%81%8e...
```

これをデコードしてファイル名にし、そのままビルドしたところ ——

| WordPress の URL | Astro が生成した URL |
| --- | --- |
| `一色産うなぎの白焼き**、**はじめました` | `一色産うなぎの白焼きはじめました` ← 読点が消えた |
| `貝のはなし**-―-**赤貝とみる貝` | `貝のはなし**--**赤貝とみる貝` ← ダッシュが消えた |

**Astro の glob loader が既定でスラッグを整形（記号を除去）していました。**

見た目には何も壊れていないため、目視では絶対に気づけません。このまま公開していたら、**10本ぶんの検索順位と外部リンクを失っていました。**

修正は `content.config.ts` の1行です。

```ts
const blog = defineCollection({
  loader: glob({
    base: './src/content/blog',
    pattern: '**/*.{md,mdx}',
    // ファイル名をそのまま ID にする（既定だと記号が削られて URL が変わる）
    generateId: ({ entry }) => entry.replace(/\.mdx?$/, ''),
  }),
  // ...
});
```

そのうえで、**WordPress の全スラッグと生成物を機械的に突き合わせ**ました。

```
=== URL 照合（WordPress vs Astro） ===
  一致   一色産うなぎの白焼き、はじめました
  一致   春の魚のはなし ― さわらが旬です
  ...
  一致: 10 / 不一致: 0
```

:::message alert
移行で最も重要なのは **URL を1文字も変えないこと**です。「なんとなく動いている」ではなく、**旧 URL の一覧と新 URL の一覧を突き合わせて全件確認**してください。ここはAIに明示的に指示する必要があります。
:::

### 8. デザインを WordPress に揃える

「見た目が変わると気づかれる」という懸念に対しては、**WordPress が実際に配信している CSS からデザイントークンを抜き出して写す**方法をとりました。目視で色を合わせたのではありません。

Twenty Twenty-Five の場合：

```
--wp--preset--color--base: #FFFFFF
--wp--preset--color--contrast: #111111
--wp--preset--font-family--manrope: Manrope, sans-serif
--wp--style--global--content-size: 645px
body { font-weight: 300; line-height: 1.4; letter-spacing: -0.1px; }
```

Astro 側では、Manrope を **Astro のフォント機能で自己ホスト**しました。Google Fonts の CDN を叩かず、ビルド時にダウンロードして自分のサーバーから配信します。

```js
// astro.config.mjs
fonts: [
  {
    provider: fontProviders.google(),
    name: 'Manrope',
    cssVariable: '--font-manrope',
    weights: [300, 400, 700],
    fallbacks: ['Hiragino Kaku Gothic ProN', 'Noto Sans JP', 'sans-serif'],
  },
]
```

```astro
<!-- src/components/BaseHead.astro -->
<Font cssVariable="--font-manrope" preload />
```

:::message
Astro の fonts 設定だけでは `@font-face` が出力されません。**`<Font>` コンポーネントを head に置く必要があります。** ここで一度ハマりました。
:::

![Astro で表示した記事ページ。WordPress のテーマと同じ書体・本文幅になっている](/images/wordpress-to-astro-vol124/astro-article.png)
*記事ページ。背景・文字色・本文幅 645px・Manrope 300 まで WordPress 側の配信 CSS に合わせている*

### 9. `/blog` を差し替える

WordPress を `blog-old-wordpress/` に退避し、Astro の成果物を `blog/` に配置しました。

そして **旧 URL 10件すべてに対して HTTP ステータスを確認**しました。

```
=== 旧 WordPress の URL がそのまま生きているか ===
  OK   一色産うなぎの白焼き、はじめました       200
  ...
  生存: 10 件 / 失敗: 0 件
```

#### ここで事故りました

上記の「10件すべて 200」は、**記事ページだけを個別に叩いた結果**でした。一覧ページのリンクを辿っていなかったため、**記事一覧のリンクが `/blog/blog/` になっている**ことに気づかず、参加者からの指摘で発覚しました。

原因は、`base: '/blog'` にしたのにページの置き場所が `src/pages/blog/` のままだったことです。

```
base: '/blog'  +  src/pages/blog/index.astro  →  /blog/blog/
```

WordPress と同じ構造（`/blog/` が一覧、`/blog/<記事名>/` が本文）に合わせるため、`src/pages/blog/[...slug].astro` を `src/pages/[...slug].astro` に移動し、`src/pages/blog/index.astro` を削除しました。

**教訓**：個別 URL の疎通確認だけでは足りません。**一覧ページの HTML からリンクを全部抜き出して、片端からアクセスする**必要があります。

```
一覧ページのリンクを全部たどって、切れているものがないか確認してください
```

「動いているように見える」と「本当に全部動いている」は別物で、後者は**確かめ方を指定しないと出てきません**。

---

## 発展：Cloudflare Pages へ追い出す

ここまでで「動くプログラム」は消えましたが、ブログのファイルは EC-CUBE と同じサーバーに残っています。**さらに一段、安全にできます。**

```
                    ┌ /blog/*  → Cloudflare Pages（HTMLのみ）
お客様 → Cloudflare ┤
                    └ それ以外 → VPS の EC-CUBE
```

前提は「ドメインが Cloudflare の proxy 配下（オレンジの雲マーク）にあること」だけで、プランは Free でも始められます。

### 手順

**1. Pages にデプロイ**

`base: '/blog'` でビルドした成果物を `/blog/` 配下に置いた形でアップロードします。

```bash
mkdir -p .pages-deploy/blog && cp -r dist/. .pages-deploy/blog/
npx wrangler pages project create example-blog --production-branch=main
npx wrangler pages deploy .pages-deploy --project-name=example-blog --branch=main
```

**2. Worker で `/blog/*` を中継**

```js
// worker/index.js
const PAGES_HOST = 'example-blog.pages.dev';

export default {
  async fetch(request) {
    const url = new URL(request.url);
    url.hostname = PAGES_HOST;
    url.protocol = 'https:';
    url.port = '';
    const res = await fetch(new Request(url, request));
    const out = new Response(res.body, res);
    out.headers.set('x-served-by', 'cloudflare-pages');
    return out;
  },
};
```

**3. ルートを2本割り当てる**

```
example.shop/blog
example.shop/blog/*
```

:::message alert
`example.shop/blog*`（スラッシュなしのワイルドカード1本）にすると、**`/blog-old-wordpress` まで巻き込みます**。必ず2本に分けてください。
:::

### Origin Rules ではなく Worker を選んだ理由

Origin Rules でも理屈上は振り分けられますが、**転送先が `pages.dev`（＝Cloudflare 自身）になる構成は公式にサポートされていません**。実際、デモ中に Pages のエイリアスが一時的に **522（接続タイムアウト）** を返す場面がありました。Worker のほうが確実です。

### 効果の検証

`/blog/` に5回、EC-CUBE 側に2回アクセスして、オリジンの Apache アクセスログを確認しました。

```
=== オリジン(VPS)に届いたリクエスト: 2 件 ===
    GET /
    GET /products/list
```

**ブログへの5回は、1件もオリジンに届いていません。**

### Cloudflare proxy を有効にしたら実IPを復元する

proxy を有効にすると、オリジンから見た接続元が Cloudflare の IP になります。`mod_remoteip` で実IPを復元しておくと、EC-CUBE のログにも WordPress のコメントにも正しい IP が残ります。

```apache
# /etc/httpd/conf.d/00-cloudflare-remoteip.conf
RemoteIPHeader CF-Connecting-IP
RemoteIPTrustedProxy 173.245.48.0/20
RemoteIPTrustedProxy 103.21.244.0/22
# ... https://www.cloudflare.com/ips-v4 / ips-v6 の全レンジ
```

:::message
`https://www.cloudflare.com/ips-v4` は**末尾に改行がありません**。`cat ips-v4 ips-v6` で連結すると最終行と先頭行がくっつき、レンジを2つ取りこぼします。実際にこれで `httpd -t` が構文エラーになりました。
:::

---

## 発展：ブログ記事を商品詳細ページに埋め込む

参加者の方から「ブログ記事を EC-CUBE の商品IDに紐付けて、商品詳細ページに表示したい」というリクエストをいただき、その場で実装しました。

### 設計

```
記事の先頭に商品IDを書く
    ↓
Astro が商品IDごとに JSON を静的生成
    ↓
EC-CUBE の商品ページが JavaScript で読んで表示
```

**サーバー側で動くプログラムは1つも増えません。** 「API」の実体はただの静的 JSON ファイルです。

### 1. 記事に商品IDを持たせる

```ts
// src/content.config.ts
schema: ({ image }) =>
  z.object({
    title: z.string(),
    description: z.string(),
    pubDate: z.coerce.date(),
    heroImage: z.optional(image()),
    // EC-CUBE の商品ID
    products: z.array(z.number()).optional(),
  }),
```

```markdown
---
title: 'チェリーアイスサンドができるまで'
pubDate: 'Aug 01 2026'
products: [2]        ← これだけ
---
```

### 2. 商品IDごとの JSON を静的生成する

```ts
// src/pages/related/[id].json.ts
import type { APIRoute } from 'astro';
import { getCollection } from 'astro:content';

export async function getStaticPaths() {
  const posts = await getCollection('blog');
  const ids = new Set<number>();
  posts.forEach((p) => (p.data.products ?? []).forEach((id) => ids.add(id)));
  return [...ids].map((id) => ({ params: { id: String(id) } }));
}

export const GET: APIRoute = async ({ params }) => {
  const id = Number(params.id);
  const base = import.meta.env.BASE_URL.replace(/\/$/, '');
  const posts = (await getCollection('blog'))
    .filter((p) => (p.data.products ?? []).includes(id))
    .sort((a, b) => b.data.pubDate.valueOf() - a.data.pubDate.valueOf())
    .map((p) => ({
      title: p.data.title,
      description: p.data.description,
      url: `${base}/${p.id}/`,
      pubDate: p.data.pubDate.toISOString().slice(0, 10),
    }));
  return new Response(JSON.stringify({ productId: id, posts }, null, 2), {
    headers: { 'content-type': 'application/json; charset=utf-8' },
  });
};
```

生成される JSON（`https://example.shop/blog/related/2.json`）：

```json
{
  "productId": 2,
  "posts": [
    {
      "title": "チェリーアイスサンドができるまで",
      "description": "北海道産チェリーのアイスを、サクサクのクッキーで挟みました。",
      "url": "/blog/cherry-ice-sand/",
      "pubDate": "2026-07-31"
    }
  ]
}
```

### 3. EC-CUBE 側は JavaScript だけ

EC-CUBE 4 の **管理画面 > コンテンツ管理 > JS管理**（実体は `html/user_data/assets/js/customize.js`）に置きます。全ページで読み込まれ、**店舗オーナー自身が管理画面から編集できる**のが利点です。

```js
(function () {
  'use strict';

  var m = location.pathname.match(/\/products\/detail\/(\d+)/);
  if (!m) return;                       // 商品詳細ページ以外では何もしない
  var productId = m[1];

  var anchor = document.querySelector('.ec-productRole__description');
  if (!anchor) return;

  fetch('/blog/related/' + productId + '.json', { cache: 'no-cache' })
    .then(function (res) { return res.ok ? res.json() : null; })
    .then(function (data) {
      if (!data || !data.posts || data.posts.length === 0) return;

      var box = document.createElement('div');
      box.className = 'blog-related';
      var html = '<h2>この商品について書いた記事</h2><ul>';
      data.posts.forEach(function (p) {
        html += '<li><a href="' + p.url + '">' + esc(p.title) + '</a>'
             +  '<p>' + esc(p.description) + '</p></li>';
      });
      box.innerHTML = html + '</ul>';
      anchor.parentNode.insertBefore(box, anchor.nextSibling);
    })
    .catch(function () { /* 記事が無い商品では何も起きない */ });

  function esc(s) {
    return String(s).replace(/[&<>"']/g, function (c) {
      return { '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c];
    });
  }
})();
```

### この構成の良いところ

| | |
| --- | --- |
| **EC-CUBE の改造がゼロ** | プログラムは1行も触っていません。JS管理は管理画面から編集できます |
| **CORS 設定が不要** | ブログも商品ページも同一オリジンです。ドメインを分けていたら設定が必要でした |
| **記事が無い商品では何も起きない** | 対応する JSON が 404 を返し、JS は静かに終了します |
| **「API」がただの静的ファイル** | サーバーで動くプログラムは増えていません |

![EC-CUBE の商品詳細ページ。商品説明の下に「この商品について書いた記事」としてブログ記事へのリンクが表示されている](/images/wordpress-to-astro-vol124/product-embed.png)
*商品説明の直下に、Astro のブログ記事が差し込まれた状態*

記事側で `products: [2, 5, 9]` と並べれば、それぞれの商品ページに同じ記事が出ます。

---

## 撤去とバックアップ

最後に WordPress を撤去しました。**ここが一番大事なところです。**

フォルダ名を変えて退避しただけの状態を確認すると——

```
/blog-old-wordpress/wp-login.php   →  200  ← ログイン画面が生きている
/blog-old-wordpress/wp-admin/      →  302
```

**退避＝減っていません。** 攻撃するプログラムは「`/blog` にあるか」ではなく「**PHP が実行できるか**」しか見ていません。

そこで、**バックアップを取り、その中身を検証してから**削除しました。

```bash
BK=/home/user/backup/wp-$(date +%Y%m%d-%H%M%S)
mkdir -p $BK
tar czf $BK/wordpress-files.tar.gz blog-old-wordpress
mysqldump --single-transaction --routines --triggers example_wp > $BK/example_wp.sql

# 検証（ここを省略しない）
tar tzf $BK/wordpress-files.tar.gz > /dev/null && echo "tar OK"
grep -c 'wp_posts' $BK/example_wp.sql
```

バックアップは**公開ディレクトリの外**に置きます。Web から取得できてしまっては本末転倒です。

検証後に削除：

```bash
rm -rf blog-old-wordpress
mysql -e 'DROP DATABASE example_wp; DROP USER "example_wp"@"localhost";'
```

結果：

| | 撤去前 | 撤去後 |
| --- | --- | --- |
| `/blog-old-wordpress/wp-login.php` | **200** | **404** |
| WordPress のデータベース | あり | **削除** |
| ブログの旧URL | 10件 | **10件すべて生存** |
| EC-CUBE | 正常 | **正常** |

---

## 正直な話：これで解決しないこと

良いことばかり書いたので、最後に正直なところを。

### できなくなること

- 記事内のコメント欄（別サービスが必要）
- サイト内検索（別の仕組みで代替）
- 問い合わせフォーム（EC-CUBE 側の機能や別サービスへ）
- ブラウザだけで記事を書く運用

### これでは守れないこと

- EC-CUBE 本体の更新をサボること
- 管理画面のパスワードが弱いこと
- サーバーの設定不備
- FTP の ID とパスワードの使い回し

### 二重管理は発生する

お店側（EC-CUBE）とブログ側（Astro）で、ロゴやメニューを2か所で持つことになります。**これは Astro の欠点です。**

ただし「メニューを直すとき2か所さわる手間」と「お客様のカード情報が漏れるリスク」を天秤にかければ、答えは明らかだと考えています。実際、デザインを揃える作業もリンクの一括修正も、AIに任せられました。

**Astro 化は「扉を1つ減らす」施策**であって、家の戸締まりそのものが要らなくなるわけではありません。ただ、**いちばん破られやすい扉を消せる**のは確かです。

---

## まとめ

今日たどり着いた状態を整理します。

| 段階 | ブログの正体 | サーバーで動くもの |
| --- | --- | --- |
| 開始時 | WordPress | PHP + MySQL + 管理画面 |
| 中盤 | Astro（VPS上） | **なし**（HTMLのみ） |
| 最終 | **Astro（Cloudflare Pages）** | **なし＋VPSにファイルすら無い** |

そして **URL は1文字も変わっていません**。

### AIエージェントに任せられること・任せられないこと

**任せられた**のは、記事の執筆、Markdown 変換、リンクの一括修正、CSS トークンの抽出と適用、URL の全件照合、といった**機械的で網羅性が求められる作業**です。人間がやると必ず取りこぼす領域で、明確に強みがありました。

**任せられなかった**のは、**検証の設計**です。この記事で書いたとおり、「個別 URL は 200 だが一覧のリンクは壊れている」という状態を、AI は「全件 OK」と報告しました。**何をもって OK とするかは、人間が決めて指示する必要があります。**

冒頭の注意点に戻りますが、クレデンシャルの扱い、サーバー接続の安全性、そして**エンジニアによる監視**。この3つが揃って初めて、この記事の内容は実務で使えるものになります。

### 使ったAIエージェントとプラン（2026年8月時点）

| ツール | 提供元 | 必要なプラン（月額） |
| --- | --- | --- |
| Claude Code | Anthropic | $20〜（Pro） |
| Codex | OpenAI | $20〜（ChatGPT Plus） |
| Cursor | Cursor | $20〜（Pro） |
| GitHub Copilot | GitHub | $10〜（Pro） |
| Gemini CLI | Google | $19.99〜（AI Pro） |

条件は「**自分のパソコンの中でファイルを作り、ビルドして、アップロードまでできること**」。ブラウザで文章を書いてくれるだけのAIでは足りません。

今回の内容であれば、**月20ドル前後のプランが1つあれば足ります**。上位プランは使用量の上限を広げるためのもので、ブログを書いて公開する程度なら最安プランで十分です。

:::message
価格・プラン内容は頻繁に変わります。契約前に各社の公式サイトでご確認ください。
:::

---

## 参考リンク

- スライド：[【初心者向け】Astro でEC-CUBEのコンテンツ管理を安全にしよう](https://claude.ai/code/artifact/de61b1b5-7660-4ca2-8435-19aa5de90407)
- [Astro 公式ドキュメント](https://docs.astro.build/)
- [Cloudflare Pages](https://developers.cloudflare.com/pages/)
- [EC-CUBE 4 開発ドキュメント](https://doc4.ec-cube.net/)
- 前回（vol.123）：[エージェンティックコマース搭載 EC-CUBE4.4 最新情報](https://claude.ai/code/artifact/cfef942a-8a2e-41b7-8d2f-9ec793f485e2)
