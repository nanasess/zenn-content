---
title: "複数リポジトリの並列開発が劇的に楽になるツールを作った — git-worktree-manager"
emoji: "🌳"
type: "tech"
topics: ["git", "bash", "claudecode", "cli", "worktree"]
published: true
---

:::message
この記事は Claude Code で執筆しています。
:::

## はじめに

マイクロサービスや分離されたフロントエンド/バックエンド構成など、**1つのプロジェクトに複数の git リポジトリ**がある環境で開発していると、こんな悩みはありませんか？

- タスクを並列で進めたいが、**各リポジトリで手動でブランチを切って worktree を作る**のが面倒
- worktree をリポジトリ内に作ると、**コード検索にノイズが入る**（特に AI コーディングツールで致命的）
- worktree 作成後に **依存関係のインストールやフック実行を毎回忘れる**
- 終わったタスクの **worktree とブランチの掃除** が面倒

これらを解決するために [git-worktree-manager](https://github.com/nanasess/git-worktree-manager) を作りました。

## git-worktree-manager とは

**1コマンドで、プロジェクト配下の全リポジトリに対して worktree の作成・一覧・削除を一括で行うツール**です。

Pure Bash で書かれており、`git` と `bash` 以外の依存はありません。単一リポジトリでも複数リポジトリでも動作します。

## インストール

```bash
git clone https://github.com/nanasess/git-worktree-manager.git ~/git-repos/git-worktree-manager
ln -sf ~/git-repos/git-worktree-manager/worktree ~/.local/bin/worktree
```

## 基本的な使い方

### worktree の作成

```bash
cd ~/git-repos/my-project
worktree create feature-login
```

これだけで、プロジェクト配下の**全リポジトリ**に対して以下が実行されます：

1. `git fetch origin` でリモート更新
2. デフォルトブランチをベースに worktree とブランチを作成
3. `CLAUDE.md` に worktree コンテキスト（タスク名・作業ディレクトリ等）を付加して生成
4. 非 git アイテム（設定ファイル等）をシンボリックリンク
5. `.worktreerc` の `post_create()` フックを実行
6. lock ファイルに基づいて依存関係を自動インストール

### worktree のレイアウト

ポイントは、**worktree がプロジェクトの外に配置される**ことです：

```
~/git-repos/
├── my-project/                     # 元のプロジェクト
│   ├── frontend/                   #   git リポジトリ
│   ├── backend/                    #   git リポジトリ
│   └── CLAUDE.md
│
├── my-project.worktrees/           # worktree はプロジェクトの外
│   ├── feature-login/
│   │   ├── frontend/               #   branch: feature-login
│   │   ├── backend/                #   branch: feature-login
│   │   └── CLAUDE.md               #   worktree コンテキスト付き
│   └── fix-auth/
│       └── ...
```

リポジトリ内に worktree を作ると、IDE のファイルツリーや AI ツールのコード検索にノイズが入ってしまいますが、外に配置することでこの問題を回避できます。

### その他のコマンド

```bash
# 一覧表示
worktree list

# 全リポジトリを一括 pull
worktree pull

# 全リポジトリを一括チェックアウト
worktree checkout main

# worktree とブランチを削除
worktree cleanup feature-login --force --delete-branches

# マージ済みタスクを自動検出して一括削除
worktree cleanup --merged --force --delete-branches
```

## Claude Code との連携

このツールを作った最大の動機は、**Claude Code のサブエージェント並列処理**です。

Claude Code では `--add-dir` オプションで作業ディレクトリを指定できます。worktree を使えば、**各サブエージェントが独立したブランチで同時に作業**でき、ブランチの衝突を気にする必要がありません。

### Worktree Context

`worktree create` で作成された worktree の `CLAUDE.md` には、自動的に以下のコンテキストが付加されます：

```markdown
# Worktree Context

This directory was created by `worktree create feature-login` as a working worktree.

- **Task name**: feature-login
- **Working directory**: /home/user/git-repos/my-project.worktrees/feature-login
- **Project root (source)**: /home/user/git-repos/my-project

> **Important**: All code changes must be made within this directory.
> Do not modify the project root directly.
```

これにより、Claude Code の `/compact` でコンテキストが圧縮された後も、**エージェントが作業ディレクトリを見失わない**ようになっています。

### Claude Code Skills

```bash
worktree install --skills
```

を実行すると、Claude Code のスラッシュコマンドとして以下が使えるようになります：

- `/worktree-create <task-name>` — worktree の一括作成
- `/worktree-list` — 一覧表示
- `/worktree-cleanup <task-name>` — 一括削除
- `/worktree-checkout [branch]` — 一括チェックアウト
- `/worktree-pull` — 一括 pull

## `.worktreerc` フック

プロジェクトルートに `.worktreerc` を配置して `post_create()` 関数を定義すると、worktree 作成後に自動実行されます。

```bash
# .worktreerc
post_create() {
    # 共有ドキュメントをリンク
    ln -sf shared-docs/claude-repository-guide.md CLAUDE.md
    # 環境設定ファイルをコピー
    cp .env.example .env
}
```

利用可能な環境変数：

| 変数 | 内容 |
|---|---|
| `WORKTREE_TASK_NAME` | タスク名 |
| `WORKTREE_TASK_DIR` | タスクディレクトリのフルパス |
| `WORKTREE_PROJECT_ROOT` | 元のプロジェクトルートのフルパス |

## 単一リポジトリでも使える

単一リポジトリにも対応しています。プロジェクトルート自体が git リポジトリの場合、worktree はタスクディレクトリに直接作成されます：

```
├── my-app/                         # 単一 git リポジトリ
│   ├── src/
│   └── CLAUDE.md
├── my-app.worktrees/
│   └── feature-login/              # worktree (branch: feature-login)
│       ├── src/
│       └── CLAUDE.md               #   worktree コンテキスト付き
```

## 類似ツールとの比較

| ツール | 対象 | 言語 | 特徴 |
|---|---|---|---|
| **git-worktree-manager** | 単一/複数リポ | Bash | フック、依存関係自動インストール、Claude Code 連携 |
| [etz](https://github.com/etz-dev/etz) | 複数リポ | Node.js | GUI あり、npm パッケージ |
| [git-worktree-runner](https://github.com/coderabbitai/git-worktree-runner) | 単一リポ | Bash | AI 並列開発向け |
| [worktrunk](https://github.com/max-sixty/worktrunk) | 単一リポ | Rust | AI エージェントワークフロー向け |

git-worktree-manager は **Pure Bash で依存ゼロ**、**複数リポジトリの一括管理**、**`.worktreerc` フックによるカスタマイズ**が特徴です。

## まとめ

git-worktree-manager は、複数リポジトリの worktree 管理という地味だけど面倒な作業を自動化するツールです。

- **1コマンドで全リポジトリに worktree を作成・削除**
- **リポジトリの外に配置**してコード検索のノイズを排除
- **依存関係の自動インストール**とフックで初期セットアップを自動化
- **Claude Code との連携**で AI 並列開発をスムーズに

特に Claude Code でサブエージェントを活用した並列開発をしている方には、ぜひ試していただきたいです。

https://github.com/nanasess/git-worktree-manager
