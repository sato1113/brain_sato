# MASAKI OS

会議・AI作業・業務データ・リマインド・資料を一つにつなぐ、個人業務基盤（仮称 **MASAKI OS**）。

単なるタスク管理ではなく、「会議で決まったこと」「自分がAIで行った作業」「プロジェクト別の進捗」「自分に確認が必要な事項」までを1画面で把握できる状態を目指す。

## 確定した設計判断

| 項目 | 決定 |
|---|---|
| アーキテクチャ | 自前フル構築（Next.js + Supabase + Vercel） |
| 利用範囲 | 当面は自分専用（認証は自分のGoogleアカウント1つに限定） |
| 会議データ源 | tl;dv を正とする（Zoom API 自前実装はしない） |
| AI作業履歴 | 会話全文ではなく、要約・成果物・作業履歴を業務単位で保存 |
| 初回に作る縦串 | **会議 → 議事録 → タスク/決定事項の抽出** |
| 定期実行 | Vercel Cron（当面）。将来 Cloud Scheduler 等に拡張可 |

## ドキュメント

| ファイル | 内容 |
|---|---|
| [docs/requirements.md](docs/requirements.md) | 要件定義（全体像・スコープ・非機能要件） |
| [docs/architecture.md](docs/architecture.md) | 実装方針（技術構成・連携・認証・自動化区分・セキュリティ） |
| [docs/data-model.md](docs/data-model.md) | データモデル（テーブル一覧＋初回縦串のDDL） |
| [docs/slice-01-meeting-to-tasks.md](docs/slice-01-meeting-to-tasks.md) | 初回縦串の詳細仕様（フロー・フォーマット・受け入れ条件） |
| [docs/samples/](docs/samples/) | 実データで生成した議事録・抽出結果のサンプル |

## 開発フェーズ

- **Phase 0**：業務整理（プロジェクト分類・自動化レベル・KPI定義） … 一部を要件定義に反映済み
- **Phase 1（MVP）**：本リポジトリの当面のゴール。初回縦串＋プロジェクト/タスク管理＋ホームダッシュボード
- **Phase 2**：会議完全自動連携・Calendar/Gmail連携・週次レポート・認定パートナー/CSのKPI
- **Phase 3**：横断検索・会議前ブリーフィング・自然言語での状況質問（AIエージェント化）
- **Phase 4**：スライド/台本/音声/動画生成

## 開発ブランチ

`claude/masaki-os-design-yrvqrt`
