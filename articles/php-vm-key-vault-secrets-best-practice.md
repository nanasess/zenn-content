---
title: "『本番では Secrets Manager を使え』を PHP で実現するベストプラクティス"
emoji: "🔐"
type: "tech"
topics: ["php", "azure", "aws", "secrets", "security"]
published: true
---

:::message
この記事は Claude Code で執筆しています。
:::

## TL;DR

- **本番では Secrets Manager / Key Vault を使え**と言われるが、**ベストプラクティスを示す公式ドキュメントが PHP には存在しない** (AWS / Azure ともに)
- **PHP は二級市民**扱いで、公式 caching ライブラリも提供されていない (Go / Java / .NET / Python には存在する)
- AWS は 2024 年 7 月リリースの **Secrets Manager Agent** (Rust 製 localhost sidecar) で「**リモート API call の per-request overhead 解消**」までは実現 (ただし PHP 側は依然リクエストごとに curl 呼び出しが必要)
- **Azure には同等品が存在しない**ため、VM + Apache (PHP-FPM / mod_php どちらも) 構成では自力で配布パイプラインを組む必要がある
- 本記事では **Managed Identity → systemd EnvironmentFile → Apache `SetEnv` → `$_SERVER`** という配布パターンを、実機検証結果と共に解説する
- `getenv()` / `putenv()` は thread-safety 問題で Symfony が default deprecation 済 → **`$_SERVER` 経由で読むのが正解**
- 設計上、`/run/` (tmpfs) 配置 + `0600 root:root` で persistent disk に平文を残さない

## 前提要件 (この記事が対象とするケース)

本記事のパターンは、以下の **5 つの要件をすべて同時に満たす** ことを目的に設計している。どれか 1 つでも不要であれば、もっとシンプルな選択肢 (例: `.env` 平文 + `getenv()`) で済む。

### 1. 極力ファイルに書き出さない

- `.env` / `.env.local` / `.env.local.php` / `config.php` / `wp-config.php` に **平文で credential を残さない**
- これらに書き出してしまうと **「Secrets Manager / Key Vault を使う意味」が半減**する:
  - デプロイアーティファクト (Docker image / git archive / rsync 同期先) に残る
  - VM のバックアップ / スナップショットに含まれる
  - 開発者の手元 / 共有 dev VM に複製される
- → secret の物理痕跡は **tmpfs (`/run/`) に限定** し、persistent disk には 1 バイトも残さない

### 2. オーバーヘッドを極限まで少なくしたい

- リクエストごとに Vault REST API を叩く: **数十〜数百 ms のレイテンシ → NG**
- リクエストごとに localhost HTTP (Secrets Manager Agent) を叩く: **1〜5 ms → 改善はされるが完全にゼロではない**
- env 経由 (`$_SERVER` メモリ参照): **ほぼ 0 ms (syscall すら発生しない) → 最速**
- → リクエストパス上の HTTP 呼び出しは **完全に排除** したい

### 3. 1Password で credential を一元管理する

- 開発者ローカル / CI (GitHub Actions 等) / 本番 VM すべてで **source of truth を 1 つに集約**
- 環境ごとに別の管理ツール (`.env`、HashiCorp Vault、Kubernetes Secrets、各種クラウド Vault) を使い分けると **同期ミス / 更新漏れ / 値の不整合** が高頻度で起きる
- → 1Password を起点に Terraform → クラウド Vault → VM、と一方向の同期パイプラインを作る

### 4. `phpinfo()` から credential を見られないように注意

- 本記事の方式では credential が `$_SERVER` に入る → **`phpinfo()` を呼ぶと全部表示される**
- 過去事例: ファイル名を推測しづらく (`aaaaaaa.php` 等) しても、内部監査 / 攻撃者 / 開発者ミスで参照されると一発で漏洩
- 対策:
  - **本番では `phpinfo()` を絶対に呼ばない** (デバッグ用も含めデプロイから除外)
  - 開発環境で `phpinfo()` を使う場合は **即削除**、または `<Files>` で `Require ip ...` 制限
  - WAF / IDS で `phpinfo` 文字列を含むレスポンスを検出 / ブロック

### 5. CLI (`bin/console` 等) と HTTP リクエスト処理の両方をカバー

- Web (Apache + php-fpm / mod_php) と CLI (`bin/console doctrine:migrations:migrate` 等) で **同じ credential 取得方法** を使いたい
- vhost `SetEnv` は HTTP リクエスト処理時のみ有効 → CLI には届かない
- 解決: **CLI 実行時は `op run --env-file=...` で 1Password から env を inject**
- これにより Web と CLI 双方で `$_SERVER` / `getenv()` から同じ credential を取得可能 (詳細は後述 [CLI 経由の場合](#cli-経由の場合-op-run))

## 問題: PHP と Secrets Manager / Key Vault は相性が悪い

### PHP のプロセスモデル

PHP-FPM / mod_php は **per-request プロセスモデル**で、リクエスト処理が終わるとアプリケーション状態がリセットされる。

```
[php-fpm worker]
  Request 1: スクリプト起動 → メモリ確保 → 終了 (静的変数リセット)
  Request 2: スクリプト起動 → メモリ確保 → 終了
  ...
```

長命プロセス言語 (.NET / Java / Node.js / Python の worker / Go) であれば:
1. 起動時に Secrets Manager から取得
2. メモリにキャッシュ
3. リクエスト中はメモリから即時取得

が成立する。**PHP では「起動時 1 回」が成立しない**ため、

- リクエストごとに Vault REST API を叩く → **レイテンシ致命的** (数十〜数百 ms)
- APCu や Redis にキャッシュする → 自前実装 + 暗号化処理が必要
- 起動時に env vars として注入する → これが本記事のテーマ

という選択肢になる。

### AWS は公式 caching ライブラリを提供している? — **No**

AWS SDK PHP の `SecretsManagerClient` には **caching 機能が組み込まれていない**。Go / Java / .NET / Python には `aws-secretsmanager-caching-*` という公式 caching ライブラリが提供されているが、**PHP 版は存在しない** (2026 年 5 月時点)。

公式 docs ([`Aws\SecretsManager\SecretsManagerClient`](https://docs.aws.amazon.com/aws-sdk-php/v3/api/class-Aws.SecretsManager.SecretsManagerClient.html)) のメソッド一覧 / コンストラクタオプションにも `cache` / `ttl` / `memoize` を含むものは無し。AWS re:Post でも長年 PHP caching リクエストが上がっているが未実装。

> `CredentialProvider::memoize` は IAM 認証情報のキャッシュであり、Secrets Manager の secret value とは別物

### では AWS はどう解決しているか — **Secrets Manager Agent (2024-07)**

[`aws/aws-secretsmanager-agent`](https://github.com/aws/aws-secretsmanager-agent) という Rust 実装の **localhost HTTP sidecar** が 2024 年 7 月にリリースされている。

```
[PHP-FPM worker] --curl--> http://localhost:2773 <--cache--> AWS Secrets Manager
                            ↑
                       TTL 300s で in-memory cache
                       SSRF 防止用 token
```

PHP 側は `curl http://localhost:2773/secretsmanager/get?secretId=...` で取得する。**言語非依存**の sidecar pattern により、リモート API call の per-request overhead は解消される。

ただし注意点として、**PHP のリクエストごとに curl 呼び出しが必要**な点は変わらない (PHP-FPM / mod_php とも):

| 方式 | 1 リクエスト辺りのオーバーヘッド | rotation 対応 |
|---|---|---|
| **PHP SDK で AWS API 直叩き** | 数十〜数百 ms (network + auth + processing) | 自動 |
| **Secrets Manager Agent (cache hit)** | 1〜5 ms (localhost HTTP + JSON parse) | 自動 (TTL 内) |
| **env 注入 + `$_SERVER`** | ほぼ 0 ms (メモリ参照、syscall なし) | 手動 (httpd restart) |

→ Agent は **rotation を自動化したい / container 環境** であれば最適だが、**rotation 頻度が低い VM 構成** では env 注入の方が strictly 高速。本記事で扱うのは env 注入パターン (Agent が無い Azure 環境への適用も含む)。

### Azure Key Vault の場合 — **同等品が存在しない**

- **Microsoft 公式 PHP SDK** (`Azure/azure-sdk-for-php`) は **2023-11-27 に Archived 化** (read-only)
- Microsoft Learn の Key Vault クライアントライブラリ一覧 (.NET / Python / Java / Spring / Node.js / Go) に **PHP の行が無い**
- Azure 版の Secrets Manager Agent 相当 sidecar は提供されていない
- 公式ガイダンスは「**REST API を直接叩け**」と「メモリにキャッシュせよ」

つまり Azure VM + PHP では:
- 自前で REST クライアントを書く
- 自前で caching 層を書く
- 自前で配布パイプラインを書く

を全部やる必要がある。これが「**本番では Key Vault を使え**」という指導と現実のギャップ。

### 既存 PHP コミュニティパッケージの状況

Packagist / GitHub を調査した限り (確認時点):

| パッケージ | 内容 | 評価 |
|---|---|---|
| `Azure/azure-sdk-for-php` | 公式旧 SDK | Archived (公式メンテ終了) |
| `keboola/azure-key-vault-php-client` | コミュニティ実装、PHP 8.2+ / MI 対応 | caching 無し |
| `wapacro/az-keyvault-php` | コミュニティ実装、MI 対応 | caching 無し、メンテ停滞気味 |
| Symfony / Laravel bundle 系 | 小規模、メンテ脆弱 | 本番投入は時期尚早 |

→ **公式 SDK 不在 + コミュニティ製はどれも caching 未実装**。

## アーキテクチャの選択肢

VM + PHP (PHP-FPM / mod_php) 構成で credential を扱う pragmatic な選択肢は 3 つに集約される:

| 方式 | 概要 | 評価 |
|---|---|---|
| **A. `wp-config.php` / `.env` / `config.php` への平文ハードコード** | 従来のレガシー構成 | git に入れないなら現実的だが、デプロイアーティファクトに残る |
| **B. 起動時 env 注入 (本記事のテーマ)** | systemd / cloud-init で credential を env vars として httpd に渡す | バランス良好、persistent file 回避可能 |
| **C. リクエストごとに Vault 取得** | PHP コード内で SDK を呼ぶ | レイテンシ問題、PHP に caching 層が無いので非推奨 |

**B が現代的な事実上の標準**で、AWS の Secrets Manager Agent も「Agent が caching + 配布を担い、PHP は localhost を叩くだけ」という意味で同じ思想。

## 採用するアーキテクチャ (Azure VM + Apache 例、PHP-FPM / mod_php とも適用可)

```
[Source of truth]
   1Password (人間が管理する Vault)
        ↓ Terraform 同期 (TF_VAR_* に op read で注入)
[Cloud secrets store]
   Azure Key Vault (RBAC + soft delete + purge protection)
        ↓ VM の Managed Identity で読み取り (静的 credential 不要)
[VM 起動時 fetch]
   systemd oneshot: fetch-secrets.sh
   /run/(service)/secrets.env (tmpfs, 0600 root:root)
        ↓ systemd EnvironmentFile= で httpd プロセスに inject
[httpd プロセス env]
   APP_DB_PASSWORD / APP_SMTP_PASSWORD 等
        ↓ Apache vhost SetEnv "${APP_DB_PASSWORD}" → FastCGI params
[PHP リクエスト処理]
   $_SERVER['DB_PASSWORD'] = "..."
```

各層の役割と「なぜそうするか」を順に説明する。

## 実装詳細

### 1. Terraform で Key Vault に同期

```hcl
module "key_vault" {
  source             = "./modules/key_vault"
  resource_group     = data.azurerm_resource_group.rg
  location           = var.location
  name               = "myapp-secrets"
  tenant_id          = data.azurerm_client_config.current.tenant_id
  admin_principal_id = azuread_service_principal.app.object_id
  secrets = {
    "APP-DB-PASSWORD"   = var.app_db_password
    "APP-SMTP-PASSWORD" = var.app_smtp_password
  }
}

# VM の user-assigned MI に Key Vault Secrets User role を付与
resource "azurerm_role_assignment" "vm_kv_secrets_user" {
  scope                = module.key_vault.id
  role_definition_name = "Key Vault Secrets User"
  principal_id         = azurerm_user_assigned_identity.vm.principal_id
}
```

**ポイント**:
- **`rbac_authorization_enabled = true`** (legacy access_policy は使わない)
- **`purge_protection_enabled = true`** (誤削除防止)
- **user-assigned Managed Identity** を採用 (system-assigned だと、既存 VM に identity を追加する際 plan 時に `principal_id` が null になり role assignment が失敗する chicken-and-egg がある)
- Terraform 実行 SP に **`Key Vault Administrator` role** を付与 + **`time_sleep` で RBAC 伝播を待つ** (Azure RBAC は API 完了後にもデータプレーン反映に追加時間がかかる)

```hcl
resource "azurerm_role_assignment" "admin" {
  scope                = azurerm_key_vault.main.id
  role_definition_name = "Key Vault Administrator"
  principal_id         = var.admin_principal_id
}

resource "time_sleep" "wait_for_admin_rbac" {
  depends_on      = [azurerm_role_assignment.admin]
  create_duration = "60s"
}

resource "azurerm_key_vault_secret" "secrets" {
  for_each     = nonsensitive(toset(keys(var.secrets)))
  name         = each.key
  value        = var.secrets[each.key]
  key_vault_id = azurerm_key_vault.main.id
  depends_on   = [time_sleep.wait_for_admin_rbac]
}
```

これを入れないと apply 時に **「secret 作成リクエストが 403 Forbidden」** で失敗する。

### 2. VM 起動時の fetch script (azure-cli 非依存)

azure-cli は AlmaLinux 10 上で **「`ModuleNotFoundError: No module named 'distutils'`」**で実行時に失敗する (Microsoft 公式 RPM の bundled Python 3.6 の依存問題)。

このため **curl + REST API + system Python 3.12** で実装する:

```bash
#!/bin/bash
set -euo pipefail

readonly VAULT_NAME="myapp-secrets"
readonly VAULT_URI="https://${VAULT_NAME}.vault.azure.net"
readonly API_VERSION="7.4"
readonly OUTPUT_FILE="/run/httpd/secrets.env"
readonly TMPFILE="${OUTPUT_FILE}.tmp"

# IMDS endpoint から MI access token を取得
TOKEN=$(curl -sS -H "Metadata: true" \
    "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https%3A%2F%2Fvault.azure.net" \
    | python3 -c "import json,sys; print(json.load(sys.stdin)['access_token'])")

if [ -z "${TOKEN:-}" ]; then
    echo "fetch-secrets: failed to acquire MI token" >&2
    exit 1
fi

# 実際にこのファイルを open() するのは systemd PID 1 (root) のみ。
# httpd master / worker は env を fork 継承するので、apache uid 読み取り権は不要。
install -m 0600 -o root -g root /dev/null "$TMPFILE"

{
    for secret_url in $(curl -sS -H "Authorization: Bearer $TOKEN" \
        "${VAULT_URI}/secrets?api-version=${API_VERSION}" \
        | python3 -c "import json,sys; print('\n'.join([s['id'] for s in json.load(sys.stdin).get('value', [])]))"); do
        secret_name="${secret_url##*/}"
        env_name=$(echo "$secret_name" | tr '-' '_')
        value=$(curl -sS -H "Authorization: Bearer $TOKEN" \
            "${secret_url}?api-version=${API_VERSION}" \
            | python3 -c "import json,sys; print(json.load(sys.stdin)['value'])")
        printf "%s='%s'\n" "$env_name" "$value"
    done
} >> "$TMPFILE"

mv "$TMPFILE" "$OUTPUT_FILE"
echo "fetch-secrets: $(wc -l < "$OUTPUT_FILE") secrets written"
```

**ポイント**:
- IMDS endpoint (`169.254.169.254`) は VM 内からのみアクセス可能、認証情報不要
- Secret 値を **single quote で囲む** (systemd の特殊文字解釈を抑制)
- atomic mv で書き出し (httpd 起動と race しない)
- **`/run/` (tmpfs)** に書く = persistent disk に平文を残さない (= 廃棄ディスク / バックアップから secret が漏れない)

### 3. systemd unit

```ini
# /etc/systemd/system/fetch-secrets.service
[Unit]
Description=Fetch secrets from Azure Key Vault to tmpfs
After=network-online.target
Wants=network-online.target
# httpd 起動前に完了させる
Before=httpd.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/fetch-secrets.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/systemd/system/httpd.service.d/secrets.conf (drop-in)
[Service]
EnvironmentFile=/run/httpd/secrets.env
```

```
# /etc/tmpfiles.d/httpd-secrets.conf
d /run/httpd 0750 root apache - -
```

**ポイント**:
- `Type=oneshot` + `Before=httpd.service` で httpd 起動前に env file 生成を保証
- httpd の drop-in で **`EnvironmentFile=`** を追加 (httpd プロセスの env に展開)
- `/run/httpd/` ディレクトリは `0750 root:apache` (httpd の PID file 等の場所と整合)、その配下の `secrets.env` は `0600 root:root` (最小権限)

### 4. Apache vhost で SetEnv

```apache
# /etc/httpd/conf.d/app.example.com-le-ssl.conf
<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerName     app.example.com
    DocumentRoot   /var/www/html/app.example.com/html

    # systemd EnvironmentFile から取り込んだ env を vhost-level で php に渡す
    SetEnv DB_PASSWORD   "${APP_DB_PASSWORD}"
    SetEnv SMTP_USER     "apikey"
    SetEnv SMTP_PASSWORD "${APP_SMTP_PASSWORD}"

    Include /etc/letsencrypt/options-ssl-apache.conf
    SSLCertificateFile /etc/letsencrypt/live/app.example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/app.example.com/privkey.pem
</VirtualHost>
</IfModule>
```

**ポイント**:
- `${VAR}` は Apache の **起動時 env 展開** (httpd プロセス env を参照)
- 同じ env を複数 vhost で **別名で渡す** ことで「prod vhost は `APP_DB_PASSWORD`、staging vhost は `STAGING_DB_PASSWORD` を `DB_PASSWORD` として渡す」のような切り分けが可能
- `SMTP_USER` のように **literal 値** も SetEnv で渡せる (`apikey` 等の固定値)

### 5. PHP 側で `$_SERVER` から読む

```php
// config.php (EC-CUBE 2 系の例)
define('DB_PASSWORD',
    $_SERVER['DB_PASSWORD']
    ?? $_ENV['DB_PASSWORD']
    ?? throw new RuntimeException('DB_PASSWORD not set')
);
```

**なぜ `getenv()` を使わないのか?**

Symfony Dotenv コンポーネントのソースコード:

```php
private bool $usePutenv = false;  // ← デフォルト false

/**
 * @param bool $usePutenv If `putenv()` should be used to define
 *                        environment variables or not.
 *                        Beware that `putenv()` is not thread safe,
 *                        that's why it's not enabled by default
 */
```

`putenv()` は POSIX の `setenv(3)` 系で **スレッドセーフではない**ため、Symfony は default で `putenv()` を呼ばず、`$_ENV` / `$_SERVER` への書き込みのみ行う。`getenv()` は process 環境を読むため、`putenv()` で書かれた値しか取れない (= Symfony で Dotenv 経由で設定した値は `getenv()` では取れない、`$_SERVER` で取る)。

将来 RoadRunner / FrankenPHP / Swoole のような **長命プロセス + マルチスレッド** PHP 環境に移行する可能性を考えると、`$_SERVER` 優先のフォールバックチェーンが最も robust。

## CLI 経由の場合 (op run)

vhost `SetEnv` は **HTTP リクエスト処理時のみ有効** で、`bin/console` 等の CLI 実行時には `$_SERVER` に値が来ない。CLI を Web と同じ credential 取得ロジックで動かすには、別経路で env を inject する必要がある。

**解決: `op run --env-file=op.env -- <command>` で 1Password から実行時に env を注入**

### `op.env` (1Password 参照テンプレート、git commit 可)

```
# git にコミットして OK (実値ではなく op:// 参照のみ)
APP_DB_PASSWORD=op://myvault/app-db/password
APP_SMTP_PASSWORD=op://myvault/sendgrid/credential
```

### 実行例

```bash
# 開発者ローカル / Ansible / GitHub Actions のいずれでも同じ書き方
op run --env-file=op.env -- php bin/console doctrine:migrations:migrate
op run --env-file=op.env -- php artisan migrate
op run --env-file=op.env -- php data/install/install.php
```

`op run` は子プロセス (`php bin/console ...`) の env に **メモリ経由で直接 inject** する。`.env` 等の中間ファイルを作らないため:

- **ファイルシステムに値が一切残らない** (要件 1 を満たす)
- 値はプロセス memory のみに存在し、終了で消える
- 開発者 / CI 共通の手順 (要件 3 を満たす)

### CLI 側 PHP コード (Web 側と統一)

```php
$dbPassword = $_SERVER['APP_DB_PASSWORD']
    ?? $_ENV['APP_DB_PASSWORD']
    ?? getenv('APP_DB_PASSWORD')
    ?: throw new RuntimeException('APP_DB_PASSWORD not set');
```

CLI SAPI では `op run` 経由で inject された env が `$_SERVER` / `$_ENV` / `getenv()` 全てで取得可能 (検証マトリックス参照)。Web (vhost `SetEnv` → `$_SERVER` のみ) との差分を吸収するため **`$_SERVER` 優先のフォールバックチェーン** を採用。

### CI (GitHub Actions) からの実行

```yaml
- uses: 1password/load-secrets-action@v2
  env:
    OP_SERVICE_ACCOUNT_TOKEN: ${{ secrets.OP_SERVICE_ACCOUNT_TOKEN }}
    APP_DB_PASSWORD: op://myvault/app-db/password
    APP_SMTP_PASSWORD: op://myvault/sendgrid/credential

- name: Run migrations
  run: php bin/console doctrine:migrations:migrate
```

CI では `op run` ではなく [`1password/load-secrets-action`](https://github.com/1Password/load-secrets-action) を使うのが標準で、Service Account Token 経由で env を inject。1Password を起点とした **Web / CLI / CI すべてで一貫した credential 配布** が実現される (要件 3, 5 を同時に満たす)。

## AWS Secrets Manager の場合 (差分のみ)

クラウド backend を AWS Secrets Manager に置き換える場合、**systemd EnvironmentFile + Apache `SetEnv` + `$_SERVER`** の VM 側ロジックは完全に同じ。違いは以下の 3 点だけ:

1. **VM 認証**: Azure Managed Identity → **EC2 IAM Instance Profile**
2. **Secret 取得**: Key Vault REST API (Bearer token) → **AWS CLI**
   - AWS API は SigV4 署名が必要なため curl 直叩きは現実的でなく、AWS CLI を使う方が圧倒的に簡単
   - AWS CLI v2 は self-contained で Microsoft 版 azure-cli のような distutils 問題は無い
3. **インフラ定義**: Azure Key Vault リソース → **AWS Secrets Manager リソース + IAM Role/Policy**

### Terraform (AWS)

```hcl
resource "aws_secretsmanager_secret" "secrets" {
  for_each = var.secrets
  name     = "myapp/${each.key}"
}

resource "aws_secretsmanager_secret_version" "secrets" {
  for_each      = aws_secretsmanager_secret.secrets
  secret_id     = each.value.id
  secret_string = var.secrets[each.key]  # 1Password から TF_VAR_secrets に注入
}

# EC2 instance role に Secrets Manager 読み取り権限を付与
resource "aws_iam_role" "ec2" {
  name = "myapp-ec2"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy" "ec2_secrets_read" {
  role = aws_iam_role.ec2.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = "secretsmanager:GetSecretValue"
      Resource = [for s in aws_secretsmanager_secret.secrets : s.arn]
    }]
  })
}

resource "aws_iam_instance_profile" "ec2" {
  name = "myapp-ec2"
  role = aws_iam_role.ec2.name
}

resource "aws_instance" "app" {
  iam_instance_profile = aws_iam_instance_profile.ec2.name
  # ... ami / instance_type / subnet_id / etc.
}
```

### fetch-secrets.sh (AWS CLI 利用版)

```bash
#!/bin/bash
# AWS Secrets Manager から secret 群を取得し /run/httpd/secrets.env に書き出す。
# AWS CLI は IMDS 経由で IAM Instance Profile credential を自動取得するため、
# 環境変数や ~/.aws/credentials の設定は不要。
set -euo pipefail

readonly REGION="ap-northeast-1"
readonly NAME_PREFIX="myapp/"
readonly OUTPUT_FILE="/run/httpd/secrets.env"
readonly TMPFILE="${OUTPUT_FILE}.tmp"

# 実際にこのファイルを open() するのは systemd PID 1 (root) のみ。
install -m 0600 -o root -g root /dev/null "$TMPFILE"

{
    for secret_name in $(aws secretsmanager list-secrets \
        --region "$REGION" \
        --filters "Key=name,Values=$NAME_PREFIX" \
        --query 'SecretList[].Name' --output text); do
        # myapp/db_password → DB_PASSWORD (prefix 除去 + 大文字化)
        env_name=$(echo "$secret_name" | sed "s|^${NAME_PREFIX}||" | tr 'a-z-/' 'A-Z__')
        value=$(aws secretsmanager get-secret-value \
            --region "$REGION" \
            --secret-id "$secret_name" \
            --query SecretString --output text)
        printf "%s='%s'\n" "$env_name" "$value"
    done
} >> "$TMPFILE"

mv "$TMPFILE" "$OUTPUT_FILE"
echo "fetch-secrets: $(wc -l < "$OUTPUT_FILE") secrets written"
```

### systemd / Apache / PHP 部分

**Azure 例と完全に同じ** — `fetch-secrets.service` の oneshot unit、httpd の EnvironmentFile drop-in、vhost の `SetEnv "${VAR}"`、PHP の `$_SERVER['VAR']` 取得ロジック、すべて再利用可能。クラウド backend に依らない設計の利点。

### Secrets Manager Agent との比較

AWS は前述の通り **Secrets Manager Agent** (localhost sidecar) という選択肢もある。VM + Apache 構成で本記事のパターンと Agent パターンを比較:

| 観点 | env 注入 (本記事) | Secrets Manager Agent |
|---|---|---|
| 1 req のオーバーヘッド | ~0 ms (メモリ) | 1〜5 ms (localhost HTTP) |
| ローテーション | 手動 (httpd restart) | 自動 (TTL 300s 以内に反映) |
| PHP コードの変更 | `$_SERVER` 読み出しのみ | curl + JSON parse 関数を全リクエストで呼ぶ |
| 追加プロセス | なし | Agent (Rust) が常駐 |
| 環境横断 | AWS / Azure / 任意クラウド | AWS 限定 |

**ローテーション頻度が低く / マルチクラウド対応したい** → 本記事の env 注入パターン
**ローテーション自動化を最優先 / AWS 限定で OK** → Secrets Manager Agent

## 実機検証マトリックス (env 経路 × 取得方法)

実機 (docker-compose で再現テスト) での挙動:

| SAPI | env 注入元 | `$_SERVER` | `$_ENV` | `getenv()` |
|---|---|---|---|---|
| mod_php (`php:apache`) | docker-compose `environment:` | **✗** | ✓ | ✓ |
| mod_php (`php:apache`) | Apache `SetEnv` | **✓** | ✗ | ✓ |
| php-fpm (`php:fpm` + nginx) | docker-compose `environment:` | ✓ | ✓ | ✓ |
| php-fpm (`php:fpm` + Apache Event MPM + mod_proxy_fcgi) | docker-compose `environment:` (FPM container) | ✓ | ✓ | ✓ |
| php-fpm + Apache | Apache **`SetEnv`** (mod_proxy_fcgi 経由) | **✓** | ✓ | ✓ |
| php-fpm + Apache | Apache **`PassEnv`** | ✗ | — | — |
| CLI | プロセス env (`op run --env-file=` 経由) | ✓ | ✓ | ✓ |

**重要な発見**:

1. **`$_SERVER` がすべてのケースで動く唯一の経路** (mod_php + docker-compose env を除く)
2. **`PassEnv` は mod_proxy_fcgi では FastCGI params に乗らない** (`SetEnv` を使う必要がある)
3. **php-fpm 公式 Docker image (`php:fpm`) は `clear_env = no` がデフォルト設定済** — 一方、素の RPM/deb パッケージは `clear_env = yes` がデフォルトなので、`/etc/php-fpm.d/www.conf` で明示的に切る必要あり

### Apache process と FPM process は別 — env 伝達経路に注意

```
[Apache process]    env = X  ← Apache の environ
       │
       │ mod_proxy_fcgi で FastCGI 接続
       │ (FastCGI params だけが送られる)
       ↓
[FPM worker process]   env = Y  ← 独立した environ、Apache の env は見えない
```

**Apache に env を渡しても FPM worker は見えない**。FPM worker に env を渡したいなら:

- `/etc/php-fpm.d/www.conf` に `env[X] = $X` + `clear_env = no` を書く (FPM 側 env 注入)
- または vhost の `SetEnv` を使って **FastCGI params 経由** で渡す (こちらが推奨)

`SetEnv` は **mod_proxy_fcgi が FastCGI params として送る** ため、FPM worker は受け取って `$_SERVER` に展開する。

## セキュリティ考察

### `/run/` (tmpfs) 配置の意味

| 防げる脅威 | 効果 |
|---|---|
| VM image スナップショット流出 | ✓ tmpfs はスナップショットに含まれない |
| バックアップツールが secret を吸い上げ | ✓ `/run/` は通常バックアップ対象外 |
| 廃棄ディスクからの forensics 復元 | ✓ tmpfs は RAM only |
| **VM への apache user 経由侵入** | ✗ `/proc/<httpd-worker-pid>/environ` から読める |
| **CI / git からの漏洩** | ✓ Ansible テンプレートに secret を埋め込まない |

### `0600 root:root` の意義

`/run/httpd/secrets.env` を実際に open() するのは **systemd PID 1 (root) だけ**。httpd master (root) が EnvironmentFile を読み、worker (apache uid) は fork で env を継承する。

→ **apache uid に read 権限を与える必要はない** (least privilege)。

attack surface の差:
- `0600 root:root`: apache uid 取得時も `cat /run/httpd/secrets.env` で読めない
- `0640 root:apache`: apache uid 取得時に**ファイル経由で読める**

ただし apache uid を取れば `/proc/self/environ` で env から secret は読める (= 同じ値が見える) ため、**実害は限定的**。それでも **least privilege** に従う価値はある。

### ローテーション

Key Vault で secret 値を更新した後:

```bash
sudo systemctl restart fetch-secrets.service  # /run/httpd/secrets.env 再生成
sudo systemctl restart httpd                  # EnvironmentFile 再読み込み (reload では不十分)
```

**Apache の `reload` は EnvironmentFile を再読しない** (graceful reload は config file のみ再読)。env 変更時は必ず `restart` が必要。in-flight request が失敗する可能性はあるため、ローテーション時はメンテナンスウィンドウを設けるか blue-green デプロイで対応。

AWS Secrets Manager + RDS の自動ローテーションに相当する仕組みは、PHP では実装が複雑になるため、本パターンでは **手動 / 半自動ローテーション** が現実解。

### 設計上、secret が露出する瞬間

このパターンを採用しても、以下の場所では secret が平文で存在する:

1. **Apache プロセス memory** (`/proc/<pid>/environ`)
2. **PHP リクエスト処理中の `$_SERVER`** (= memory)
3. **tmpfs 上の `/run/httpd/secrets.env`** (短時間、root only)
4. **`phpinfo()` 出力 — 最も注意が必要** (下記参照)

→ どんな方式でも runtime 中はメモリに存在する。**「persistent disk に平文を残さない」 + 「git / CI に平文を入れない」** が実質的なゴール。

### `phpinfo()` は本パターン最大の漏洩リスク

:::message alert
本パターンでは credential が `$_SERVER` に入るため、**`phpinfo()` を呼ぶと credential が HTML としてそのままレンダリングされる**。本パターン固有の致命的な弱点で、以下の対策を **必ず** 実施すること。
:::

| 環境 | 対策 |
|---|---|
| 本番 | `phpinfo()` を呼ぶスクリプトを **デプロイから完全除外**。`opcache.preload` / 静的解析 (PHPStan / Psalm) で `phpinfo` 呼び出しを検知 / ブロック |
| staging / 開発 | `phpinfo` 用ファイルは **使用後即削除** + `.htaccess` で `<Files>` ディレクティブで IP 制限 |
| CI / 共有 dev | `phpinfo` を含む PR は **コードレビューで必ず指摘** + branch protection で main にマージ不可に |
| ログ / WAF | 出力に `_SERVER\['` パターンや credential らしき長文字列が含まれた応答を **検知 → アラート / ブロック** |
| 動作確認時 | `phpinfo` 出力を grep / cat / less 等で参照する際は **値そのものを表示せず length / 先頭数文字のみマスク表示**。ログ / コンソール / chat transcript への意図せぬ流出を防ぐ |

## このパターンが向く / 向かないケース

### 向く

- **VM ベース** の Apache + PHP (PHP-FPM / mod_php とも)
- **マルチサイト同居** (vhost ごとに別 secret を設定可能)
- **EC-CUBE 2 / 4 / レガシー WordPress / 旧式 Symfony** など、env-based config に refactor 可能なアプリ
- **ローテーション頻度が低い** credential (年次更新程度)

### 向かない

- **頻繁な自動ローテーション** が要求される (AWS Secrets Manager + RDS rotation 等) → Secrets Manager Agent + retry-on-auth-fail パターン、または RDS Proxy + IAM 認証への移行を検討
- **ハードコード前提のレガシー PHP** (`wp-config.php` / `.env` / `config.php` 改修不可) → 別の妥協が必要
- **App Service / コンテナ環境** → Key Vault references / ECS task definition `secrets:` 等のネイティブ機能を使うべき

## 残課題と展望

### Azure 側

- **Azure 版 Secrets Manager Agent** が出てくれば、本パターンは不要になる可能性
- 現状の Microsoft 公式アナウンスでは予定なし

### PHP 側

- **`RoadRunner` / `FrankenPHP`** で long-running PHP プロセスが普及すれば、Symfony と同じ「起動時 fetch + メモリ caching」が成立する
- ただし mod_php / php-fpm の per-request モデルからの移行コストは大きい

### このパターン自体

- **複数 VM への展開**: 各 VM が独立して KV から取得 → 自然に整合
- **ローテーション自動化**: Key Vault の event 通知 → Functions → VM への restart trigger、で半自動化可能
- **本パターンを Ansible モジュール化** して再利用可能に

## まとめ

- **「本番では Secrets Manager / Key Vault を使え」**という指導には、PHP では現実的なギャップがある
- AWS は 2024 年に **Secrets Manager Agent** という言語非依存の sidecar を提供し、リモート API call の per-request overhead は解消 (ただし localhost への curl は依然必要)。Azure には同等品が無いため、自力で配布パイプラインを組む必要がある
- **systemd EnvironmentFile + Apache `SetEnv` + `$_SERVER`** の組み合わせで、persistent disk に平文を残さず、`getenv()` / `putenv()` の thread-safety 問題も回避できる
- `mod_php` / `php-fpm` / CLI で挙動が異なるため、実機検証と環境マトリックスの理解が必須
- VM 構成のレガシー PHP アプリにおける現代的な credential 管理パターンとして、本記事のアプローチは再現可能

### 参考リンク

- [AWS Secrets Manager Agent (公式リポジトリ)](https://github.com/aws/aws-secretsmanager-agent)
- [Aws\\SecretsManager\\SecretsManagerClient (公式 API リファレンス)](https://docs.aws.amazon.com/aws-sdk-php/v3/api/class-Aws.SecretsManager.SecretsManagerClient.html)
- [Azure/azure-sdk-for-php (Archived)](https://github.com/Azure/azure-sdk-for-php)
- [Microsoft Learn - Client Libraries for Azure Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/general/client-libraries)
- [Symfony Configuration (公式ドキュメント)](https://symfony.com/doc/current/configuration.html)
- [Symfony Dotenv ソースコード (`usePutenv = false`)](https://github.com/symfony/symfony/blob/7.4/src/Symfony/Component/Dotenv/Dotenv.php)
- [PHP-FPM Configuration (`clear_env` / `env[]`)](https://www.php.net/manual/en/install.fpm.configuration.php)
