# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## このリポジトリについて

[Zenn](https://zenn.dev) の記事・本を管理するコンテンツリポジトリ。zenn-cli を使って記事を作成・プレビューし、GitHub連携でZennに公開する。

## よく使うコマンド

```sh
# 依存関係のインストール
mise run install

# 新しい記事を作成
mise run create-article

# プレビューサーバーを起動（ブラウザで http://localhost:8000 を開く）
pnpm exec zenn preview

# フォーマットチェック
mise run check-format

# フォーマット適用
mise run format
```

## 記事フォーマット

`articles/` 以下の各 `.md` ファイルは以下のフロントマターを持つ：

```yaml
---
title: "記事タイトル"
emoji: "😊" # 記事のアイキャッチ絵文字（1文字）
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["topic1"] # タグ（最大5つ）
published: false # true にすると公開される
publication_name: genda_jp # 所属publication（省略可）
---
```

## ディレクトリ構成

- `articles/` — 記事ファイル（ファイル名がスラッグになる）
- `books/` — 本のディレクトリ
