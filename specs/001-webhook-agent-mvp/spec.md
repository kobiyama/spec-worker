# Feature Specification: Webhook Agent MVP

**Feature Branch**: `001-webhook-agent-mvp`
**Created**: 2025-01-24
**Status**: Draft
**Input**: User description: "GitHub Issueが作成されたらWebhookを受信し、Claude Codeエージェントを実行してPRを自動作成するMVP。P1機能: Issue作成時の自動処理、処理状況のIssueコメント通知、エラー時の通知。認証はエージェントに委譲。EC2に手動デプロイ。1リポジトリ固定。"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Issue作成からPR自動生成 (Priority: P1) 🎯 MVP

開発者がGitHub Issueを作成すると、システムが自動的にIssueの内容を解析し、AIエージェント（Claude Code）を起動してコード変更を行い、Pull Requestを作成する。

**Why this priority**: これがサービスの核心的価値。Issue作成からPR作成までの自動化フローが動作しなければ、他の機能は意味を持たない。

**Independent Test**: Issueを1つ作成し、数分後にPRが自動作成されることを確認する。PRにはIssueへのリンクと変更内容が含まれる。

**Acceptance Scenarios**:

1. **Given** リポジトリにWebhookが設定されている状態で、**When** 開発者が新しいIssueを作成する、**Then** システムは10秒以内にWebhookを受け付け、バックグラウンドでエージェント処理を開始する
2. **Given** エージェントがIssueの内容を処理中のとき、**When** エージェントがコード変更を完了する、**Then** システムは変更内容をコミットし、PRを作成する
3. **Given** PRが作成されたとき、**When** 開発者がPRを確認する、**Then** PRのタイトルと説明にはIssueの情報が含まれ、関連Issueへのリンクがある

---

### User Story 2 - 処理状況のIssueコメント通知 (Priority: P1)

システムがIssueを処理している間、進捗状況をIssueのコメントとして投稿し、ユーザーが処理状況を把握できるようにする。

**Why this priority**: ユーザーがシステムの動作を確認できることは、信頼性と透明性の観点から必須。処理が数分〜30分かかる可能性があるため、フィードバックなしでは不安を与える。

**Independent Test**: Issueを作成し、「処理開始」「PR作成完了」のコメントがIssueに自動投稿されることを確認する。

**Acceptance Scenarios**:

1. **Given** システムがWebhookを受信したとき、**When** エージェント処理を開始する、**Then** 「処理を開始しました」という旨のコメントをIssueに投稿する
2. **Given** エージェント処理が完了したとき、**When** PRが作成される、**Then** 「PRを作成しました」という旨のコメントとPRへのリンクをIssueに投稿する

---

### User Story 3 - エラー時の通知 (Priority: P1)

エージェント処理中にエラーが発生した場合、ユーザーにエラー内容を通知し、手動対応の判断ができるようにする。

**Why this priority**: エラーが発生してもユーザーに通知されなければ、Issue作成後に何も起こらず混乱を招く。デバッグと手動復旧のためにエラー情報は必須。

**Independent Test**: 意図的にエラーを発生させ（例：不正なIssue形式）、Issueにエラーコメントが投稿されることを確認する。

**Acceptance Scenarios**:

1. **Given** エージェント処理中にエラーが発生したとき、**When** システムがエラーを検知する、**Then** エラー内容を含むコメントをIssueに投稿する
2. **Given** Webhookの署名検証が失敗したとき、**When** システムが不正なリクエストを拒否する、**Then** リクエストは処理されず、ログに記録される（Issueへのコメントは不要）
3. **Given** エージェント処理がタイムアウトしたとき、**When** 設定時間を超過する、**Then** タイムアウトエラーをIssueコメントで通知し、処理を中断する

---

### Edge Cases

- Issueが非常に長い内容（10,000文字以上）の場合、エージェントへの入力をどう扱うか？（合理的なデフォルト：そのまま渡す。エージェント側で処理）
- Issueが空の本文で作成された場合はどうするか？（合理的なデフォルト：タイトルのみでエージェント処理を試行、失敗時はエラー通知）
- 同じIssueに対して複数回Webhookが送られた場合はどうするか？（合理的なデフォルト：Issue番号で重複チェック、処理中なら無視）
- エージェントがコード変更なしで終了した場合はどうするか？（合理的なデフォルト：「変更なし」としてIssueにコメント通知、PRは作成しない）
- サーバー再起動時に処理中のジョブがあった場合はどうするか？（合理的なデフォルト：ログに記録し、手動再トリガーを促す）

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: システムはGitHub IssueのWebhook（issues.opened イベント）を受信できなければならない（MUST）
- **FR-002**: システムはWebhookの署名をHMAC-SHA256で検証し、無効な署名は401で拒否しなければならない（MUST）
- **FR-003**: システムはWebhook受信後10秒以内に200 OKを返却し、処理は非同期で実行しなければならない（MUST）
- **FR-004**: システムはIssueの内容（タイトル・本文・番号・リポジトリ情報）をエージェントに渡せなければならない（MUST）
- **FR-005**: システムはエージェント（Claude Code CLI）を実行し、処理完了を待機できなければならない（MUST）
- **FR-006**: システムはエージェントが作成したコミットをPull Requestとして作成できなければならない（MUST）
- **FR-007**: システムは処理開始時にIssueへコメントを投稿できなければならない（MUST）
- **FR-008**: システムはPR作成完了時にIssueへコメント（PRリンク含む）を投稿できなければならない（MUST）
- **FR-009**: システムはエラー発生時にIssueへエラー内容を含むコメントを投稿できなければならない（MUST）
- **FR-010**: システムは各リクエストを独立した作業ディレクトリで処理しなければならない（MUST）
- **FR-011**: システムは同一Issueの重複処理を防止しなければならない（MUST）
- **FR-012**: システムはすべての操作を構造化ログとして記録しなければならない（MUST）
- **FR-013**: システムは機密データ（トークン、シークレット）をログに出力してはならない（MUST NOT）

### Key Entities

- **Job**: 1つのWebhookリクエストに対応する処理単位。Issue番号、リポジトリ情報、処理状態（pending/processing/completed/failed）、開始時刻、相関IDを持つ
- **Webhook Payload**: GitHubから受信するIssueイベントのデータ。action、issue（number、title、body）、repository（full_name、clone_url）を含む
- **Agent Result**: エージェント実行の結果。成功/失敗フラグ、コミット有無、エラーメッセージを含む

### Assumptions

以下はユーザー入力で明示されなかったが、合理的なデフォルトとして採用した前提：

- 対象リポジトリは1つに固定（環境変数で指定）
- GitHub認証トークンはエージェント側の設定に依存（サーバーはPR作成用トークンのみ必要）
- エージェントのタイムアウトは環境変数で設定可能（デフォルト30分）
- 並列処理数の上限は設けない（サーバーリソースに依存）
- Issueの言語は問わない（エージェントが対応）
- PRのブランチ名は `feature/issue-{番号}` 形式

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Issueが作成されてから10秒以内にシステムがWebhookを受け付け、処理開始コメントがIssueに投稿される
- **SC-002**: 正常なIssueに対して、エージェント処理完了後5分以内にPRが作成される（エージェント処理時間を除く）
- **SC-003**: PRの成功率が90%以上である（エージェントが適切に処理できるIssue形式の場合）
- **SC-004**: すべてのエラーがIssueコメントとして通知され、ユーザーが手動対応の判断ができる
- **SC-005**: 署名検証により、不正なWebhookリクエストが100%拒否される
- **SC-006**: 10件の同時リクエストを処理しても、システムがクラッシュせず全リクエストが処理される
