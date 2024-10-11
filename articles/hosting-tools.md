---
title: "ホスティグ業務で活用しているツールやサービス"
emoji: "🔪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ansible", "terraform", "server", "Linux"]
published: true
---
*この記事は「【ショップオーナー向け】ホスティングサービスの裏側ご紹介 | EC-CUBE名古屋 vol.108」の登壇資料です*

EC-CUBEと切っても切れない関係なのがサーバーです。
EC-CUBEの開発をする傍らサーバーの運用もするのですが、運用効率向上のために様々なツールを利用しています。
今回は、EC-CUBEのホスティングサービスでどんなツールやアプリを活用し、日々の保守運用をしているかを、社会見学のようにご説明したいと思います。

## インフラ環境

- Microsoft Azure の仮想マシン(IaaS)を使用
- AlmaLinux8 または 9 を利用して、最大10年間のセキュリティサポート
    - 一部 Miracle Linux8 を利用
- 1アカウント1VMで冗長構成なし
- 基本的に B1ms(1コア2GB) インスタンスを利用。必要に応じてスケールアップ
- SSH公開鍵認証での接続のみ可能

https://azure.microsoft.com/ja-jp/products/virtual-machines/

### Infrastructure as Code(IaC)

インフラ構築・管理には Terraform 及び Ansible を使用。
テンプレートは GitHub で公開している。

https://github.com/skirnir-dev/hosting-service-terraform-template

#### Terraform

- Terraform の backend は Terraform Cloud
- APIキーなどのクレデンシャル情報は Terraform Cloud で管理
- Azure の認証は Dynamic Provider Credentials を利用
- GitHub と連携して VCS Workflow を利用
    - Pull Request で `terraform plan` を実行
    - Pull Request のマージで `terraform apply` を実行
- 以下のサービスを Terraform で管理している
    - Microsoft Azure
    - Cloudflare
    - SendGrid
    - Mackerel

https://www.terraform.io/

#### Ansible

- ミドルウェアの管理は Ansible を利用している
- terraform provider for ansible を利用して、Terraform の情報からインベントリを自動生成
- ミドルウェアの管理の他、サーバーに接続許可している公開鍵の管理もしている

https://www.ansible.com/

## 連携サービス

### SendGrid

- メール送信はこれ一択
- Pro100k プランを利用して、Teammates で各アカウントを管理
- メルマガ配信にも利用いただいている
- APIキーは Terraform で管理

https://sendgrid.kke.co.jp/

### Cloudflare

- EC-CUBE2系やEC-CUBE4.1系を利用される場合は、決済代行会社のセキュリティガイドラインをクリアできないため導入必須
- 主に Pro プランを利用
- 設定は Terraform で管理

https://www.cloudflare.com/ja-jp/

### GitHub

- ソースコード管理はこれ
- サーバーに直接 clone して利用
- 顧客がサーバー上のソースを修正しても把握できるのがメリット

https://github.com/

### Mackerel

- 外形監視に利用
- 安心の国内サービス
- 設定は Terraform で管理
- LINE に通知できるのがメリット

https://ja.mackerel.io/

## その他ツールなど

### Facebook Messenger

- 顧客連絡はほとんどこれ
- だいたい誰でも使える
- 添付ファイル容量制限なし(1ファイル100MB)
- 過去ログ閲覧制限なし
- 無料

しかし、検索とか他サービスの連携とか考えると Line Works とかの方がよいのか悩み中

### Visual Studio Code

- 顧客に SSH 経由でサーバー上のファイルを編集していただくために利用
- ファイルのパーミッションを強固に設定できるため、セキュリティが大幅に向上できる

https://azure.microsoft.com/ja-jp/products/visual-studio-code

### Emacs

- 僕が SSH 経由でサーバー上のファイルを編集するために利用
- tramp-mode と magit 最強
- Emacs のアイコンは僕のデザインです

https://www.gnu.org/software/emacs/
