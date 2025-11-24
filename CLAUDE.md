# CLAUDE.md

このファイルはClaude Codeがこのリポジトリで作業する際のガイダンスを提供します。

## コミュニケーション言語

- ユーザーとのやりとりは**日本語**で行うこと
- コード内のコメントや変数名は英語を維持
- コミットメッセージは英語（Conventional Commits形式）

## プロジェクト概要

Spec Worker - GitHub Issueのwebhookを受け取り、AIエージェント（Claude Code）に処理を委譲してPR作成まで自動化するサービス。

**アーキテクチャ**: EC2常駐サーバー + 非同期処理（GitHubの10秒タイムアウト制限対応）

## 重要なファイル

- `.specify/memory/constitution.md` - プロジェクトの憲法（原則とガバナンス）
- `.specify/templates/` - 仕様書、計画、タスクのテンプレート

## 開発ルール

- 憲法に定義された5つの原則を遵守すること
- TypeScript + Node.js 20.x を使用
- 常駐サーバー + 非同期処理アーキテクチャ
- 並列処理にはGit Worktreeを使用

## エージェント

本プロジェクトではClaude Code（Anthropic）を主エージェントとして使用する。

```bash
# 基本的な呼び出し（非対話モード）
claude --dangerously-skip-permissions -p "タスクの説明"
```

詳細な設定は `.specify/memory/constitution.md` の「エージェント設定」セクションを参照。
