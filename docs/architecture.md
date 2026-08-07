# 実装方針（MASAKI OS）

## 1. 技術構成

| 分類 | 採用 | 補足 |
|---|---|---|
| 開発 | Claude Code | 設計・実装・改修担当 |
| フロントエンド / API | Next.js（App Router, TypeScript） | 画面と API Route を同居 |
| デプロイ | Vercel | Web + Cron |
| DB | Supabase（PostgreSQL） | RLS 有効化 |
| 認証 | Google OAuth（Supabase Auth 経由） | allowlist で自分のみ許可 |
| ファイル保存 | Google Drive / Google Docs | 議事録・資料・成果物 |
| 会議 | tl;dv API | メタデータ・議事録・文字起こし取得 |
| AI処理 | Claude API（既定） | 議事録生成・抽出。OpenAI も差し替え可能に抽象化 |
| 定期実行 | Vercel Cron | 会議ポーリング・リマインド・レポート |
| ソース管理 | GitHub（本リポジトリ） | |
| エラー監視 | Sentry（Phase 2 で導入検討） | |
| グラフ | Recharts（KPI導入時） | Phase 2 |

### 重要な前提：連携はアプリが自前の資格情報を持つ

このリポジトリで開発する Next.js アプリは、Claude Code セッションの MCP 連携（tl;dv/Notion/Drive 等）を利用できない。
**デプロイするアプリは各サービスの API キー / OAuth を自前で保持する。** 必要な資格情報は §5。

## 2. 全体データフロー

```
tl;dv（録画・文字起こし・AI議事録）
  → [Cron] 新規会議をポーリング取得
  → meetings / meeting_participants に登録
  → [AI] 文字起こしから議事録を生成（目標フォーマット）
  → Google Drive / Docs に保存（artifacts にリンク登録）
  → [AI] 決定事項・タスク・確認待ちを抽出
  → decisions / action_items / confirmations に登録（要確認フラグつき）
  → ダッシュボードで表示・確認・確定
  → [Cron] 期限・確認待ちをリマインド（Gmail / Calendar）
```

AI作業履歴の流れ（F9）：

```
ChatGPT / Claude / Claude Code 等での作業
  → ダッシュボードから簡易登録（作業名＋成果物URL）／Claude Code は Hooks で送信
  → ai_work_logs に記録し、プロジェクト・成果物に紐付け
  → 後から検索・再利用
```

## 3. 自動化の3区分（最重要ポリシー）

処理は必ず次のいずれかに分類し、DBの `automations.mode` で管理する。

### 自動実行（auto）
議事録の下書き生成 / Drive格納 / タスク候補抽出 / 定期データ取得 / 数値集計 / リマインド / 週次レポート下書き / ファイル・フォルダ整理

### 承認後に実行（approval_required）
外部向けメール送信 / LINE配信 / タスク担当者への依頼送付 / 公開資料の更新 / 会員情報の変更 / 数値データの確定

### 必ず手動（manual_only）
契約・コンプライアンス判断 / 会員ランク・権限の重要変更 / 報酬確定 / 外部公開前の最終承認 / 個人情報を含むデータの削除

> 抽出された議事録・タスク・決定事項は、初期状態を「下書き/未確認（draft）」とし、
> 自分が確認して「確定（confirmed）」に変えるまで外部連携（依頼送付・メール等）は起こさない。

## 4. 認証・認可

- Supabase Auth の Google プロバイダを使用。
- ログイン許可は allowlist（`m.sato@holyday.co.jp`）。それ以外は拒否。
- Supabase の RLS を全テーブルで有効化。当面は「許可ユーザーのみ全行アクセス可」の単純ポリシーから開始し、マルチユーザー化時に owner ベースへ拡張する。

## 5. 必要な資格情報（要準備）

| # | 資格情報 | 用途 | 保管 |
|---|---|---|---|
| 1 | Claude API キー | 議事録生成・抽出 | Vercel 環境変数 / Supabase Vault |
| 2 | tl;dv API キー | 会議取得 | 同上。要：tl;dvプランでのAPI利用可否確認 |
| 3 | Google Cloud OAuth クライアント（ID/Secret） | Drive保存・Googleログイン | 同上 |
| 4 | Supabase プロジェクト（URL / anon / service_role） | DB | 同上。service_role はサーバー側のみ |
| 5 | Vercel プロジェクト | デプロイ・Cron | — |

シークレットはコミットしない。`.env.local`（ローカル）と Vercel 環境変数で管理し、`.env.example` に項目のみ記載する。

## 6. セキュリティ・機密情報の扱い

会議文字起こし・CS・会員情報は機密性が高い。以下を定義・順守する。

| 観点 | 方針 |
|---|---|
| 保存対象 | 議事録・決定・タスクは保存。文字起こし全文は必要範囲のみ保持し、原文はtl;dvリンク参照を基本とする |
| 保存期間 | 機密度に応じて設定（後続で確定）。削除手順を用意 |
| 閲覧権限 | 自分のみ。RLS＋allowlist |
| AI送信範囲 | AIへ渡すのは要約・抽出に必要な範囲。個人情報の不要な送信を避ける |
| 削除 | 個人情報削除は manual_only。削除は audit_logs に記録 |
| 操作履歴 | 承認・確定・削除・外部送信を audit_logs に記録 |
| 外部接続 | 接続先を tl;dv / Google / Claude に限定。最小権限のスコープのみ付与 |

## 7. リポジトリ構成（予定）

```
/                     … Next.js アプリ（App Router）
  app/                … 画面・API Route
  lib/
    integrations/     … tl;dv / drive / claude のクライアント（差し替え可能な抽象化）
    db/               … Supabase クライアント・型
  supabase/
    migrations/       … SQL マイグレーション
docs/                 … 本ドキュメント群
```

## 8. 環境と実行

- ローカル：`.env.local` に §5 の値を設定して `next dev`。
- 定期処理：Vercel Cron が API Route（例 `/api/cron/poll-meetings`）を叩く。Cron エンドポイントは秘密トークンで保護。
- すべての Cron 実行は `automation_runs` に開始/終了/結果/コストを記録する。
