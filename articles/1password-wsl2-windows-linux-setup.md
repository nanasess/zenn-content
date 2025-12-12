---
title: "WSL2 で 1Password for Windows と 1Password for Linux を併用する"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["1Password", "WSL2", "SSH", "Git"]
published: true
---

:::message
この記事は [Claude Code](https://docs.anthropic.com/en/docs/claude-code) に筆者の環境（`~/.gitconfig`、`~/.ssh/config`、1Password の設定ファイル等）を調査・分析させ、その結果を元に Claude Code が執筆しました。
:::

WSL2 環境で 1Password for Windows と 1Password for Linux を併用し、Git 署名は Windows Hello で簡便に、SSH 接続と CLI 連携は Linux 側で完結させる設定方法を紹介します。

## 構成の概要

| 用途 | 使用するアプリ | 認証方法 |
|------|---------------|----------|
| Git コミット署名 | 1Password for Windows | Windows Hello（生体認証） |
| SSH 接続 (WSL2) | 1Password for Linux | 1Password アプリ認証 |
| GitHub CLI / Git credential | 1Password CLI (Linux) | 1Password for Linux 連携 |

### この構成のメリット

- **Git 署名**: Windows Hello により指紋認証などで簡便に署名できる
- **SSH 接続**: WSL2 内で完結し、Linux ネイティブの SSH Agent を利用
- **CLI 連携**: 1Password for Linux と CLI が連携し、`gh` コマンドや Git credential を自動処理

:::message
npiperelay などの中継ツールは使用しません。それぞれの 1Password が独立して動作します。
:::

## アーキテクチャ

```
┌─────────────────────────────────────────────────────────────────┐
│                        Windows Host                              │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ 1Password for Windows                                       ││
│  │ - Windows Hello: 有効                                       ││
│  │ - CLI 連携: 無効                                            ││
│  └─────────────────────────────────────────────────────────────┘│
│                              │                                   │
│                              │ op-ssh-sign.exe（Git署名時のみ）  │
│                              ▼                                   │
├──────────────────────────────────────────────────────────────────┤
│                           WSL2                                   │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ 1Password for Linux                                         ││
│  │ - SSH Agent: 有効 → ~/.1password/agent.sock                ││
│  │ - CLI 連携: 有効                                            ││
│  └─────────────────────────────────────────────────────────────┘│
│                              │                                   │
│              ┌───────────────┼───────────────┐                   │
│              ▼               ▼               ▼                   │
│  ┌───────────────┐ ┌─────────────────┐ ┌───────────────────────┐│
│  │ SSH 接続      │ │ gh コマンド     │ │ Git credential        ││
│  │ (リモート)    │ │ (1Password CLI) │ │ (HTTPS認証)           ││
│  └───────────────┘ └─────────────────┘ └───────────────────────┘│
└──────────────────────────────────────────────────────────────────┘
```

## 前提条件

- Windows 11 with WSL2
- 1Password for Windows（デスクトップアプリ）
- 1Password for Linux（デスクトップアプリ、WSLg で動作）
- 1Password CLI (`op`) が WSL2 にインストール済み
- 同一の 1Password アカウントで両方にサインイン済み

## セットアップ手順

### 1. 1Password for Windows の設定

**設定 → 開発者** で以下を設定:

| 項目 | 設定値 |
|------|--------|
| SSH エージェント | 有効 |
| CLI との連携 | **無効** |

**設定 → セキュリティ** で以下を設定:

| 項目 | 設定値 |
|------|--------|
| Windows Hello でロック解除 | 有効 |

### 2. 1Password for Linux の設定

**設定 → 開発者** で以下を設定:

| 項目 | 設定値 |
|------|--------|
| SSH エージェント | 有効 |
| CLI との連携 | **有効** |

SSH Agent を有効にすると、`~/.1password/agent.sock` が自動的に作成されます。

### 3. 1Password CLI のインストール (WSL2)

```bash
# Ubuntu/Debian
curl -sS https://downloads.1password.com/linux/keys/1password.asc | \
  sudo gpg --dearmor --output /usr/share/keyrings/1password-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/1password-archive-keyring.gpg] https://downloads.1password.com/linux/debian/$(dpkg --print-architecture) stable main" | \
  sudo tee /etc/apt/sources.list.d/1password.list

sudo apt update && sudo apt install 1password-cli
```

### 4. SSH 設定

```config:~/.ssh/config
Host *
  IdentityAgent ~/.1password/agent.sock

Include ~/.ssh/conf.d/*.conf
```

### 5. シェル設定

```bash:~/.zshrc
# 1Password for Linux の SSH Agent を使用
export SSH_AUTH_SOCK=~/.1password/agent.sock

# 1Password Shell Plugins
[[ -f ~/.config/op/plugins.sh ]] && source ~/.config/op/plugins.sh
```

### 6. Git 設定

```ini:~/.gitconfig
[user]
    name = Your Name
    email = your.email@example.com
    # 1Password に保存されている SSH 公開鍵
    signingkey = ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...

[gpg]
    format = ssh

[gpg "ssh"]
    # Windows の op-ssh-sign.exe を使用（Windows Hello 連携）
    program = "/mnt/c/Users/<YOUR_USERNAME>/AppData/Local/Microsoft/WindowsApps/op-ssh-sign.exe"

[commit]
    gpgsign = true

[credential "https://github.com"]
    helper = !/usr/bin/op plugin run -- gh auth git-credential

[credential "https://gist.github.com"]
    helper = !/usr/bin/op plugin run -- gh auth git-credential
```

:::message alert
`<YOUR_USERNAME>` を実際の Windows ユーザー名に置き換えてください。
:::

### 7. GitHub CLI プラグインの設定

```bash
op plugin init gh
```

## op-ssh-sign.exe のパス

環境によって異なる場合があります:

| インストール方法 | パス |
|-----------------|------|
| Microsoft Store 版 | `/mnt/c/Users/<USERNAME>/AppData/Local/Microsoft/WindowsApps/op-ssh-sign.exe` |
| 直接インストール版 | `/mnt/c/Users/<USERNAME>/AppData/Local/1Password/app/8/op-ssh-sign.exe` |

## 動作確認

### SSH Agent の確認

```bash
ssh-add -l
# 256 SHA256:... id_ed25519 (ED25519)
```

### Git 署名の確認

```bash
# テストコミット（Windows Hello の認証ダイアログが表示される）
git commit --allow-empty -m "test: verify commit signing"

# 署名の確認
git log --show-signature -1
```

### GitHub CLI の確認

```bash
gh auth status
```

## トラブルシューティング

### SSH Agent に接続できない

```bash
# ソケットの存在を確認
ls -la ~/.1password/agent.sock

# 1Password for Linux が起動しているか確認
pgrep -f 1password
```

1Password for Linux デスクトップアプリが起動していないと、ソケットが作成されません。

### Git 署名が失敗する

1. 1Password for Windows が起動しているか確認
2. Windows Hello が有効か確認
3. `op-ssh-sign.exe` のパスが正しいか確認:
   ```bash
   ls -la "/mnt/c/Users/<USERNAME>/AppData/Local/Microsoft/WindowsApps/op-ssh-sign.exe"
   ```

### 1Password CLI が認証できない

```bash
# CLI の状態を確認
op account list
```

1Password for Linux の **設定 → 開発者 → CLI との連携** が有効になっているか確認してください。

## gh コマンドの承認プロンプトを回避する

1Password Shell Plugins を使用すると、`gh` コマンドを実行するたびに承認プロンプト（リッチ認証ダイアログ）が表示されます。これはセキュリティ上の仕様ですが、頻繁に `gh` コマンドを使用する場合は煩わしく感じることがあります。

承認プロンプトを回避したい場合は、Shell Plugins の代わりに **1Password Environments** を使用して `GITHUB_TOKEN` 環境変数を設定する方法があります。

### 1. Shell Plugins を無効化

```bash:~/.zshrc
# 1Password Shell Plugins (disabled - using 1Password Environments instead)
# [[ -f ~/.config/op/plugins.sh ]] && source ~/.config/op/plugins.sh
```

### 2. Shell Plugins のシンボリックリンクを削除

Shell Plugins は `~/bin/gh` にシンボリックリンクを作成している場合があります。これが残っていると Shell Plugins 経由の認証が使われてしまいます。

```bash
# シンボリックリンクの確認
ls -la ~/bin/gh

# シンボリックリンクの削除
rm ~/bin/gh
```

### 3. 1Password Environments で GITHUB_TOKEN を設定

1. 1Password for Linux を開く
2. **設定 → 開発者 → Environments** を開く
3. 環境を選択（または新規作成）
4. **Variables** タブで「Add Variable」をクリック
5. 名前: `GITHUB_TOKEN`、値: 1Password に保存した GitHub Personal Access Token を選択
6. **Destinations** タブで「Local .env file」を選択し、パスを指定（例: `~/.zsh/.env.local`）
7. 「Mount .env file」をクリック

### 4. シェルで .env.local を読み込む

```bash:~/.zshenv
# 1Password Environments から秘匿情報を読み込む
if [[ -f "$ZDOTDIR/.env.local" ]]; then
  set -a
  source "$ZDOTDIR/.env.local"
  set +a
fi
```

:::message
1Password Environments の Local .env file は UNIX 名前付きパイプ（FIFO）として提供されるため、プレーンテキストがディスクに保存されることはありません。
:::

### 5. Git credential helper を変更

```ini:~/.gitconfig
[credential "https://github.com"]
    helper = !gh auth git-credential

[credential "https://gist.github.com"]
    helper = !gh auth git-credential
```

### 6. 動作確認

```bash
# 新しいシェルを開いて確認
gh auth status
```

### 構成の比較

| 項目 | Shell Plugins | 1Password Environments |
|------|--------------|----------------------|
| 承認プロンプト | 毎回表示 | なし |
| トークン管理 | 1Password（都度取得） | 1Password（環境変数経由） |
| セキュリティ | 高（都度承認） | 中（起動時に読み込み） |

## 参考リンク

- [1Password SSH Agent](https://developer.1password.com/docs/ssh/)
- [1Password CLI](https://developer.1password.com/docs/cli/)
- [1Password Shell Plugins](https://developer.1password.com/docs/cli/shell-plugins/)
- [1Password Environments](https://developer.1password.com/docs/environments/)
- [Git Commit Signing with SSH Keys - GitHub Docs](https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification#ssh-commit-signature-verification)
