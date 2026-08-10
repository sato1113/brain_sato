---
name: meeting-intake
description: tl;dv の会議を MASAKI OS（Notion）に取り込む。会議→議事録生成→タスク/決定事項/確認待ちの抽出→Notion登録までを一括実行する。「会議を取り込んで」「議事録にして」「MASAKI OS に入れて」「tl;dv 取り込み」等の指示で使う。追加のAPIキー・課金は不要（コネクタのみ）。
---

# meeting-intake — 会議取り込み（MASAKI OS）

tl;dv の会議を取得し、議事録を生成して Notion に登録し、決定事項・タスク・確認待ちを抽出して登録する。実行主体は Claude（このスキルを読んだあなた）。**tl;dv / Notion コネクタのみ使用。APIキー不要。**

## 使うコネクタ
- tl;dv：`search-meetings` / `get-meeting-metadata` / `get-meeting-transcript`
- Notion：`notion-create-pages` / `notion-query-data-sources`（重複確認）/ `notion-update-page`

## Notion データソースID（登録先）
| DB | data_source_id |
|---|---|
| Projects | `e06f4cc9-79d7-4c96-b319-5ed086b700a3` |
| Meetings | `47723365-03cc-4591-bd2e-0ab98b97db61` |
| Decisions | `29b3a02c-9a83-4113-9093-85fc5e57d386` |
| Tasks | `e8409de8-a9fa-4359-bdd8-21b69441cfb5` |
| Confirmations | `5a1838ed-8fbb-4070-bcaa-526974b391c3` |
| Automation Runs | `5da6bb3b-c615-4fb6-9ec3-be8fd4e0bc39` |

MASAKI OS ホーム：https://app.notion.com/p/3b8622849bf981919dc0df4f384de454

## 手順

1. **対象決定**：指示（「直近N件」「この会議」等）に従い `search-meetings` で対象を取得。
2. **重複チェック（冪等）**：Meetings DB を `tl;dv ID` で検索し、既登録はスキップ。
3. **本文取得**：`get-meeting-transcript`。取れなければ Meetings に `議事録状態=pending` で登録し停止。
4. **プロジェクト解決**：会議名から標準プロジェクトへマッピング（下表）。該当なければ空でよい。必要なら Projects に新規作成。
5. **会議登録**：Meetings に1件作成。本文に議事録（下記フォーマット）を記載。`議事録状態=generated`。
6. **抽出＆登録**：決定→タスク→確認待ちの順で作成し、会議・プロジェクト・関連決定にリレーション（親のURLを先に取得してから子に渡す）。すべて `status=draft`（Confirmations は `open`）。
7. **記録**：Automation Runs に1件（ジョブ名・件数・結果）を記録。
8. **報告**：登録件数と、要確認(needs_review)の項目を一覧で報告。ユーザーが Notion で確認し `confirmed` にするまで外部連携はしない。

### 標準プロジェクト・マッピング（会議名→プロジェクト）
- システム会社MTG → 報酬計算システム改修
- マーケティングMTG → マーケティング
- VIGOMTG → VIGO
- バックオフィスMTG → バックオフィス
- ミラテラ✖️satoshi定例 → ミラテラ×satoshi
- 「YYYY/M/D-Meeting」等の汎用名 → 内容から判断（不明なら空）

## 議事録フォーマット（Meetings ページ本文）
```
> ⚠️ ASR由来のため数値・金額・氏名は要原文確認。

## 1. 会議の目的
## 2. 結論
## 3. 決定事項        （D1, D2… と採番）
## 4. 議論内容（要点）
## 5. タスク           （Tasks DB 参照）
## 6. 確認待ち事項
## 7. 次回までに行うこと
## 8. 関連資料
```

## 抽出スキーマ（Notion プロパティへマッピング）
- **Decisions**：内容 / コード(D1..) / 会議 / プロジェクト / 要確認 / 原文リンク / ステータス=draft
- **Tasks**：タイトル / 担当 / 自分のタスク（Masaki本人担当のみ✓）/ 期限(絶対) / 期限(原文) / 優先度 / ステータス=draft / 外部ID(WBS等) / 要確認 / 原文リンク / 会議 / プロジェクト / 関連決定
- **Confirmations**：内容 / 待ち先 / ステータス=open / 要確認 / 原文リンク / 会議 / プロジェクト

### ルール
- **期限**：会議日を基準に相対表現（「本日中」「お盆明け」等）を絶対日付へ変換。曖昧なら絶対日付は空、原文のみ。
- **要確認**：ASRで数値・金額・氏名・WBS番号が怪しい項目は `要確認=✓`。`原文リンク` に tl;dv のタイムスタンプ（`<meeting_url>?t=秒`）。
- **自分のタスク**：担当が Masaki 本人のときのみ✓。他者依頼は✓を付けず追跡対象にする。
- **リレーション値**：対象ページのURL配列で渡す。親（Project→Meeting→Decisions）を先に作成しURLを得てから子（Tasks）に渡す。
- **チェックボックス値**：`__YES__` / `__NO__`。日付は `date:<列名>:start` に ISO 文字列。

## 注意
- 会議文字起こしは機密。Notion本文には要約中心で残し、全文は tl;dv 参照（原文リンク）を基本にする。
- 大量取り込み時は新しい会議から順に処理し、各会議ごとに Automation Runs へ記録する（途中失敗しても冪等に再開できる）。
