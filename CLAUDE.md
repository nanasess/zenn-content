# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.
日本語で回答してください

## 概要

これは Zenn（日本の技術情報共有プラットフォーム）のコンテンツリポジトリです。主に EC-CUBE（日本のオープンソース EC サイト構築システム）に関する技術記事を含んでいます。

## コマンド

### 記事のプレビュー
```bash
# Docker Compose を使用（推奨）
docker-compose up

# または直接 Zenn CLI を使用
npx zenn preview
```

プレビューサーバーは http://localhost:8000 で起動します。

### 新しい記事の作成
```bash
npx zenn new:article
```

## アーキテクチャと構造

### ディレクトリ構成

- **`/articles/`** - 技術記事の Markdown ファイル
  - ファイル名は記事の slug として使用される
  - 各記事の冒頭には YAML front matter でメタデータを定義
  
- **`/images/`** - 記事で使用する画像ファイル
  - 記事ごとにサブディレクトリで整理（例: `/images/ai-customize-eccube/`）
  
- **`/docker/`** - Zenn CLI 実行用の Docker 環境設定
  - Node.js 14 ベースの Dockerfile を含む
  
- **`/books/`** - Zenn の書籍コンテンツ用（現在未使用）

### 記事の書き方

記事は Markdown 形式で、ファイルの先頭に以下のような front matter を含める必要があります：

```yaml
---
title: "記事のタイトル"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["eccube", "php", "docker"] # 5つまで
published: true
---
```

画像の参照は相対パスで行います：
```markdown
![説明テキスト](/images/記事名/画像ファイル名.png)
```

## 主な記事のトピック

このリポジトリは主に以下のトピックに関する記事を含んでいます：

1. **EC-CUBE** - 日本のオープンソース EC プラットフォーム
   - プラグイン開発
   - パフォーマンス最適化
   - セキュリティ
   - AI を使用したカスタマイズ

2. **インフラ・ホスティング**
   - Cloudflare との統合
   - Docker 環境構築
   - スケーリング戦略

3. **開発ツール・プラクティス**
   - GitHub Actions
   - E2E テスティング
   - AI コーディングアシスタント（Claude Code）

## 注意事項

- 記事の公開状態は front matter の `published` フラグで制御
- 画像ファイルは必ず `/images/記事名/` ディレクトリに配置
- EC-CUBE 関連の記事が多いため、PHP や Symfony フレームワークの知識があると理解しやすい
