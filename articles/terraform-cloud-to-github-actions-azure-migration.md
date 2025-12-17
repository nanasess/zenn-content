---
title: "Terraform Cloud から Azure Storage Backend + GitHub Actions への移行ガイド"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Terraform", "GitHubActions", "Azure", "1Password", "OIDC"]
published: true
---

:::message
この記事は [Claude Code](https://docs.anthropic.com/en/docs/claude-code) に筆者の移行作業（Issue、PR、設定ファイル等）を調査・分析させ、その結果を元に Claude Code が執筆しました。
:::

Terraform Cloud の 旧 Free プラン(5ユーザーまでならリソース数制限なし)が廃止されることに伴い、Azure Storage Backend + GitHub Actions への移行を実施しました。本記事では、その移行手順と発見事項を共有します。

## 移行の目的

- Terraform Cloud への依存を排除
- 1Password によるシークレットの一元管理
- GitHub Actions による CI/CD の実現
- 他プロジェクトへの展開が容易なモジュール化

## アーキテクチャ

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              1Password                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Vault: your-workload-identity                                        │   │
│  │  └── azure-credentials (TENANT_ID, SUBSCRIPTION_ID, CLIENT_ID)       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           GitHub Actions                                     │
│  ┌───────────────────┐     ┌──────────────────────────────────────────┐    │
│  │ Secrets           │     │ Workflow: terraform.yml                   │    │
│  │ OP_SERVICE_       │────▶│  1. 1Password からシークレット取得         │    │
│  │ ACCOUNT_TOKEN     │     │  2. OIDC で Azure 認証                    │    │
│  └───────────────────┘     │  3. Terraform plan/apply                  │    │
│                            └──────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
           │                                    │
           │ OIDC Token                         │
           ▼                                    ▼
┌─────────────────────────────┐          ┌─────────────────────────┐
│ Microsoft Entra ID          │          │ Azure Storage           │
│ App Registration            │          │ Storage Account         │
│ + Federated Credentials     │          │ (tfstate backend)       │
│ + User Access Administrator │          └─────────────────────────┘
└─────────────────────────────┘
```

## 前提条件

- Azure サブスクリプション
- Microsoft Entra ID（旧 Azure AD）へのアクセス権限
- GitHub リポジトリ
- 1Password アカウント（Business または Teams プラン）
- 既存の Terraform Cloud ワークスペース

## 移行手順

### Phase 1: GitHub Actions 用 Federated Credentials モジュール作成

GitHub Actions から Azure へ OIDC 認証するための Federated Credentials を作成するモジュールを作成します。

```hcl:modules/app_github_actions/variables.tf
variable "application_object_id" {
  description = "The object ID of the Azure AD application"
  type        = string
}

variable "display_name" {
  description = "Display name for the federated identity credential"
  type        = string
}

variable "organization" {
  description = "GitHub organization name"
  type        = string
}

variable "repository" {
  description = "GitHub repository name"
  type        = string
}

variable "environment" {
  description = "GitHub environment name"
  type        = string
  default     = "production"
}
```

```hcl:modules/app_github_actions/main.tf
resource "azuread_application_federated_identity_credential" "github_actions" {
  application_id = var.application_object_id
  display_name   = var.display_name
  description    = "GitHub Actions for ${var.organization}/${var.repository}"
  audiences      = ["api://AzureADTokenExchange"]
  issuer         = "https://token.actions.githubusercontent.com"
  subject        = "repo:${var.organization}/${var.repository}:environment:${var.environment}"
}
```

### Phase 2: Federated Credentials の作成

モジュールを使用して、GitHub Actions 用の Federated Credentials を作成します。

```hcl:main.tf
module "your_terraform_github_actions" {
  source                = "./modules/app_github_actions"
  application_object_id = azuread_application.your_app.id
  display_name          = "your-terraform-github-actions"
  organization          = "your-org"
  repository            = "your-terraform-repo"
  environment           = "production"
}
```

:::message
この段階では、まだ Terraform Cloud 経由で `terraform apply` を実行します。
:::

### Phase 3: tfstate 用 Storage Account 作成

Azure Storage に tfstate を保存するための Storage Account を作成します。

```hcl:main.tf
resource "azurerm_resource_group" "tfstate" {
  name     = "rg-your-workload-identity"
  location = "japaneast"
}

resource "azurerm_storage_account" "tfstate" {
  name                     = "yourstorageaccount"
  resource_group_name      = azurerm_resource_group.tfstate.name
  location                 = azurerm_resource_group.tfstate.location
  account_tier             = "Standard"
  account_replication_type = "LRS"

  blob_properties {
    versioning_enabled = true
  }
}

resource "azurerm_storage_container" "tfstate" {
  name                  = "tfstate"
  storage_account_name  = azurerm_storage_account.tfstate.name
  container_access_type = "private"
}

# Storage Blob Data Contributor role for Service Principal
resource "azurerm_role_assignment" "sp_storage_blob_data_contributor" {
  scope                = azurerm_storage_account.tfstate.id
  role_definition_name = "Storage Blob Data Contributor"
  principal_id         = azuread_service_principal.your_sp.object_id
}
```

:::message alert
Storage Blob Data Contributor のロール割り当てを作成するには、Service Principal に **User Access Administrator** ロールが必要です。詳細は「発見事項」セクションを参照してください。
:::

### Phase 4: 1Password 設定

1Password にシークレットを登録します。

#### 4.1 Vault の作成

1Password で新しい Vault `your-workload-identity` を作成します。

#### 4.2 シークレットの登録

Vault 内に `azure-credentials` というアイテムを作成し、以下のフィールドを追加します：

| フィールド名 | 値 |
|-------------|-----|
| `AZURE_TENANT_ID` | Microsoft Entra ID のテナント ID |
| `AZURE_SUBSCRIPTION_ID` | Azure サブスクリプション ID |
| `AZURE_CLIENT_ID` | App Registration のクライアント ID |
| `TF_VAR_tenant_id` | Terraform 変数用（AZURE_TENANT_ID と同じ値） |

#### 4.3 Service Account の作成

GitHub Actions から 1Password にアクセスするための Service Account を作成します。

1. 1Password の管理画面で **Integrations → Service Accounts** を開く
2. 「Create a Service Account」をクリック
3. 名前: `github-actions-your-org`
4. アクセス権限: 作成した Vault への **読み取り専用** アクセスを付与
5. トークンを生成してコピー

### Phase 5: GitHub 設定

#### 5.1 Repository Secret の設定

リポジトリの **Settings → Secrets and variables → Actions** で以下を設定：

| Name | Value |
|------|-------|
| `OP_SERVICE_ACCOUNT_TOKEN` | Phase 4 で生成したトークン |

#### 5.2 Environment の作成

**Settings → Environments** で `production` を作成します。

### Phase 6: GitHub Actions ワークフロー作成

```yaml:.github/workflows/terraform.yml
name: Terraform

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    environment: production
    if: github.event_name == 'pull_request'

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Load secrets from 1Password
        uses: 1password/load-secrets-action@v2
        with:
          export-env: true
        env:
          OP_SERVICE_ACCOUNT_TOKEN: ${{ secrets.OP_SERVICE_ACCOUNT_TOKEN }}
          AZURE_TENANT_ID: op://your-workload-identity/azure-credentials/AZURE_TENANT_ID
          AZURE_SUBSCRIPTION_ID: op://your-workload-identity/azure-credentials/AZURE_SUBSCRIPTION_ID
          AZURE_CLIENT_ID: op://your-workload-identity/azure-credentials/AZURE_CLIENT_ID
          TF_VAR_tenant_id: op://your-workload-identity/azure-credentials/TF_VAR_tenant_id

      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ env.AZURE_CLIENT_ID }}
          tenant-id: ${{ env.AZURE_TENANT_ID }}
          subscription-id: ${{ env.AZURE_SUBSCRIPTION_ID }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.9.0
          terraform_wrapper: false

      - name: Setup tfcmt
        uses: shirakiya/setup-tfcmt@v3

      - name: Terraform Init
        run: terraform init
        env:
          ARM_USE_OIDC: true
          ARM_CLIENT_ID: ${{ env.AZURE_CLIENT_ID }}
          ARM_TENANT_ID: ${{ env.AZURE_TENANT_ID }}
          ARM_SUBSCRIPTION_ID: ${{ env.AZURE_SUBSCRIPTION_ID }}

      - name: Terraform Format Check
        run: terraform fmt -check

      - name: Terraform Plan
        run: tfcmt plan -patch -- terraform plan -no-color
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ARM_USE_OIDC: true
          ARM_CLIENT_ID: ${{ env.AZURE_CLIENT_ID }}
          ARM_TENANT_ID: ${{ env.AZURE_TENANT_ID }}
          ARM_SUBSCRIPTION_ID: ${{ env.AZURE_SUBSCRIPTION_ID }}

  apply:
    name: Terraform Apply
    runs-on: ubuntu-latest
    environment: production
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Load secrets from 1Password
        uses: 1password/load-secrets-action@v2
        with:
          export-env: true
        env:
          OP_SERVICE_ACCOUNT_TOKEN: ${{ secrets.OP_SERVICE_ACCOUNT_TOKEN }}
          AZURE_TENANT_ID: op://your-workload-identity/azure-credentials/AZURE_TENANT_ID
          AZURE_SUBSCRIPTION_ID: op://your-workload-identity/azure-credentials/AZURE_SUBSCRIPTION_ID
          AZURE_CLIENT_ID: op://your-workload-identity/azure-credentials/AZURE_CLIENT_ID
          TF_VAR_tenant_id: op://your-workload-identity/azure-credentials/TF_VAR_tenant_id

      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ env.AZURE_CLIENT_ID }}
          tenant-id: ${{ env.AZURE_TENANT_ID }}
          subscription-id: ${{ env.AZURE_SUBSCRIPTION_ID }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.9.0
          terraform_wrapper: false

      - name: Setup tfcmt
        uses: shirakiya/setup-tfcmt@v3

      - name: Terraform Init
        run: terraform init
        env:
          ARM_USE_OIDC: true
          ARM_CLIENT_ID: ${{ env.AZURE_CLIENT_ID }}
          ARM_TENANT_ID: ${{ env.AZURE_TENANT_ID }}
          ARM_SUBSCRIPTION_ID: ${{ env.AZURE_SUBSCRIPTION_ID }}

      - name: Terraform Apply
        run: tfcmt apply -- terraform apply -auto-approve
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ARM_USE_OIDC: true
          ARM_CLIENT_ID: ${{ env.AZURE_CLIENT_ID }}
          ARM_TENANT_ID: ${{ env.AZURE_TENANT_ID }}
          ARM_SUBSCRIPTION_ID: ${{ env.AZURE_SUBSCRIPTION_ID }}
```

### Phase 7: Backend 移行

#### 7.1 providers.tf の変更

```hcl:providers.tf
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-your-workload-identity"
    storage_account_name = "yourstorageaccount"
    container_name       = "tfstate"
    key                  = "your-project.tfstate"
    use_oidc             = true
  }
}
```

#### 7.2 ステート移行

:::message alert
Terraform Cloud からの移行では `-migrate-state` オプションは使用できません。手動でステートを移行する必要があります。
:::

```bash
# 1. Terraform Cloud からステートを取得
terraform state pull > terraform.tfstate.backup

# 2. providers.tf の backend を azurerm に変更

# 3. Azure にログイン（ローカル実行用）
az login
az account set --subscription <SUBSCRIPTION_ID>

# 4. 新しい backend で初期化（ローカル認証）
terraform init -backend-config="use_oidc=false"

# 5. ステートをプッシュ
terraform state push terraform.tfstate.backup

# 6. 確認
terraform state list
```

### Phase 8: 動作確認

1. PR を作成して `terraform plan` が正常に動作することを確認
2. main にマージして `terraform apply` が正常に動作することを確認

## 発見事項

### dflook/terraform-github-actions と Azure OIDC の相性問題

当初 [dflook/terraform-github-actions](https://github.com/dflook/terraform-github-actions) を使用しようとしましたが、以下の問題がありました：

- Docker コンテナ内で OIDC 認証に必要な環境変数（`ACTIONS_ID_TOKEN_REQUEST_TOKEN` 等）が正しく渡されない
- Azure CLI がコンテナ内にインストールされていない

**解決策**: `hashicorp/setup-terraform` + `azure/login` + `tfcmt` の組み合わせに変更しました。

### hashicorp/setup-terraform の terraform_wrapper

`hashicorp/setup-terraform` の `terraform_wrapper` はデフォルトで有効になっています。これが有効だと、`GITHUB_TOKEN` を要求するなど予期しない動作が発生します。

**解決策**: tfcmt を使用する場合は `terraform_wrapper: false` を設定してください。

```yaml
- name: Setup Terraform
  uses: hashicorp/setup-terraform@v3
  with:
    terraform_version: 1.9.0
    terraform_wrapper: false  # 重要
```

### Service Principal の権限

Contributor ロールでは `Microsoft.Authorization/roleAssignments/write` 権限がないため、Storage Blob Data Contributor 等のロール割り当てを Terraform から作成できません。

**解決策**: Service Principal に **User Access Administrator** ロールを追加します。

```bash
# 手動で User Access Administrator を付与
az role assignment create \
  --assignee $(az ad sp list --display-name your-sp --query "[0].id" -o tsv) \
  --role "User Access Administrator" \
  --scope "/subscriptions/<SUBSCRIPTION_ID>"
```

その後、Terraform で import してコード管理下に置きます。

```hcl
import {
  id = "/subscriptions/<SUBSCRIPTION_ID>/providers/Microsoft.Authorization/roleAssignments/<ROLE_ASSIGNMENT_ID>"
  to = azurerm_role_assignment.sp_user_access_admin
}

resource "azurerm_role_assignment" "sp_user_access_admin" {
  scope                = data.azurerm_subscription.current.id
  role_definition_name = "User Access Administrator"
  principal_id         = azuread_service_principal.your_sp.object_id
}
```

## ローカル実行方法

GitHub Actions 環境外でローカルから Terraform を実行する場合：

```bash
# Azure にログイン
az login
az account set --subscription <SUBSCRIPTION_ID>

# 1Password から環境変数を読み込んで実行
op run --env-file=<(cat <<'EOF'
ARM_SUBSCRIPTION_ID=op://your-workload-identity/azure-credentials/AZURE_SUBSCRIPTION_ID
ARM_TENANT_ID=op://your-workload-identity/azure-credentials/AZURE_TENANT_ID
TF_VAR_tenant_id=op://your-workload-identity/azure-credentials/TF_VAR_tenant_id
EOF
) -- terraform init -backend-config="use_oidc=false"

# plan/apply
op run --env-file=<(cat <<'EOF'
ARM_SUBSCRIPTION_ID=op://your-workload-identity/azure-credentials/AZURE_SUBSCRIPTION_ID
ARM_TENANT_ID=op://your-workload-identity/azure-credentials/AZURE_TENANT_ID
TF_VAR_tenant_id=op://your-workload-identity/azure-credentials/TF_VAR_tenant_id
EOF
) -- terraform plan
```

## まとめ

Terraform Cloud から Azure Storage Backend + GitHub Actions への移行は、以下のメリットがあります：

- **コスト削減**: Terraform Cloud の有料プランが不要
- **シークレット管理の一元化**: 1Password で全てのシークレットを管理
- **柔軟な CI/CD**: GitHub Actions でカスタマイズ可能
- **OIDC 認証**: シークレットの直接保存が不要で安全

移行作業自体は慎重に行う必要がありますが、一度移行してしまえば運用は非常にシンプルになります。

## 参考リンク

- [Azure Storage Backend - Terraform](https://developer.hashicorp.com/terraform/language/settings/backends/azurerm)
- [Configuring OpenID Connect in Azure - GitHub Docs](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-azure)
- [1Password GitHub Action](https://github.com/1Password/load-secrets-action)
- [tfcmt](https://github.com/suzuki-shunsuke/tfcmt)
- [shirakiya/setup-tfcmt](https://github.com/shirakiya/setup-tfcmt)
