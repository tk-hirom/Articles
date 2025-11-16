# Zenn Articles Repository

このリポジトリは Zenn CLI で管理されているブログ記事のリポジトリです。

## リポジトリの構造

- `articles/` - Zenn の記事ファイル（Markdown 形式）
- `books/` - Zenn の本のファイル
- `images/` - 記事内で使用する画像ファイル

## 利用可能なコマンド

### 新しい記事を作成

```bash
npx zenn new:article
```

このコマンドで `articles/` ディレクトリ内に新しい記事の Markdown ファイルが作成されます。
ファイル名は自動的にランダムなスラッグで生成されます。

### 新しい本を作成

```bash
npx zenn new:book
```

このコマンドで `books/` ディレクトリ内に新しい本のフォルダが作成されます。

### 記事をローカルでプレビュー

```bash
npx zenn preview
```

ローカルサーバーが起動し、ブラウザで記事のプレビューを確認できます（通常は http://localhost:8000）。

## 記事のフロントマター

Zenn の記事には以下のようなフロントマターが必要です：

```yaml
---
title: "記事タイトル"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "prisma", "typescript"]
published: false # 公開設定（true または false）
---
```

## 画像の配置

記事内で画像を使用する場合は、`images/` ディレクトリに配置し、相対パスで参照します：

```markdown
![画像の説明](./images/image-name.png)
```

## 参考リンク

- [Zenn CLI の使い方](https://zenn.dev/zenn/articles/zenn-cli-guide)
- [Zenn のマークダウン記法](https://zenn.dev/zenn/articles/markdown-guide)
