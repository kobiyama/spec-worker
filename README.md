# Spec Worker

GitHub Issueが作成されたらWebhookを受信し、AIエージェント（Claude Code）を実行してPull Requestを自動作成するサービス。

## 概要

開発者がGitHub Issueを作成すると、システムが自動的に：

1. Webhookを受信（10秒以内に応答）
2. AIエージェント（Claude Code）がIssueの内容を解析
3. コード変更を実行
4. Pull Requestを自動作成
5. 処理状況をIssueコメントで通知

## アーキテクチャ

```
GitHub Webhook
      ↓
EC2 常駐サーバー
  ├─ POST /webhook 受信
  │    ├─ 署名検証 (HMAC-SHA256)
  │    ├─ 即座に 200 OK 返却
  │    └─ 非同期でジョブ開始
  │
  └─ 非同期ジョブ処理
       ├─ Git Worktree 作成
       ├─ Claude Code 実行
       ├─ PR 作成
       └─ Worktree 削除
```

## 技術スタック

- **ランタイム**: Node.js 20.x LTS
- **言語**: TypeScript (strict mode)
- **フレームワーク**: Express または Fastify
- **AIエージェント**: Claude Code CLI

## セットアップ

### 前提条件

- Node.js 20.x
- Claude Code CLI (`npm install -g @anthropic-ai/claude-code`)
- GitHub Personal Access Token (repo, pull_request スコープ)

### 環境変数

```bash
# GitHub
GITHUB_WEBHOOK_SECRET=your_webhook_secret
GITHUB_TOKEN=your_github_token
TARGET_REPOSITORY=owner/repo

# Agent
ANTHROPIC_API_KEY=your_anthropic_api_key
AGENT_TIMEOUT_MS=1800000  # 30分（デフォルト）

# Server
PORT=3000
```

### インストール

```bash
npm install
npm run build
npm start
```

## 開発

```bash
# 開発サーバー起動
npm run dev

# テスト実行
npm test

# リント
npm run lint
```

## ライセンス

MIT
