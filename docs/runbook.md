# 運用ランブック（MASAKI OS）

コネクタ構成の実体（Notion）と、会議取り込み〜抽出の運用手順。**追加のAPIキー・課金なし。実行は Claude セッションが担う。**

## 1. Notion 基盤（作成済み）

- ホームページ「MASAKI OS」：https://app.notion.com/p/3b8622849bf981919dc0df4f384de454
- ダッシュボード（ホームページ内のリンクドビュー）：直近の会議 / タスク（期限順）/ 担当別タスク / 確認待ち

### データソースID（相互リレーション用・Claudeが登録時に参照）

| DB | data_source_id |
|---|---|
| Projects | `e06f4cc9-79d7-4c96-b319-5ed086b700a3` |
| Meetings | `47723365-03cc-4591-bd2e-0ab98b97db61` |
| Decisions | `29b3a02c-9a83-4113-9093-85fc5e57d386` |
| Tasks | `e8409de8-a9fa-4359-bdd8-21b69441cfb5` |
| Confirmations | `5a1838ed-8fbb-4070-bcaa-526974b391c3` |
| Automation Runs | `5da6bb3b-c615-4fb6-9ec3-be8fd4e0bc39` |

## 2. 会議取り込みの手順（半自動 / Phase 1）

Claude セッションで以下を指示すると実行される（例：「直近の未取込の会議を MASAKI OS に取り込んで」）。

1. **検出**：tl;dv `search-meetings` で対象会議を取得。Notion Meetings の `tl;dv ID` と突合し、未登録のみ処理（冪等）。
2. **会議登録**：Meetings に upsert（タイトル/開催日時/主催者/時間/tl;dv URL、プロジェクトは推定 or 空）。
3. **文字起こし取得**：tl;dv `get-meeting-transcript`。無ければ `議事録状態=pending` で停止。
4. **議事録生成**：目標フォーマット（slice-01 §3）で生成し、Meetings ページ本文に記載。`議事録状態=generated`。
5. **抽出**：決定/タスク/確認待ちを抽出（slice-01 §4）し、Decisions/Tasks/Confirmations に `status=draft`（Confirmations は `open`）で登録。会議・プロジェクト・関連決定にリレーション。
6. **記録**：Automation Runs に結果を1件記録。
7. **確認**：自分が Notion で内容を確認し、ステータスを `confirmed` に更新（この操作までは外部連携しない）。

### 登録時の注意（Claude向け）
- リレーションは対象ページのURL配列で渡す（先に親→子の順で作成しURLを取得）。
- 期限は絶対日付（`期限`）と原文（`期限(原文)`）を両方入れる。会議日を基準に相対表現を解決。
- ASR起因で数値・金額・氏名が怪しい項目は `要確認=✓` にし、`原文リンク` に tl;dv のタイムスタンプURL（`?t=秒`）を入れる。
- 自分（Masaki）担当のみ `自分のタスク=✓`（他者依頼の追跡と区別）。

## 3. 検証結果（初回縦串の実証）

「システム会社MTG（2026-08-06）」を実データで取り込み、以下を登録済み（すべて draft/open）。

- Project 1（報酬計算システム改修）/ Meeting 1（議事録本文つき）
- Decisions 6（D1〜D6）/ Tasks 12（T1〜T12）/ Confirmations 2 / Automation Runs 1

→ コネクタのみ・追加費用ゼロで、会議→議事録→タスク/決定/確認待ちの縦串が成立することを確認。

## 4. Phase 2 に向けた検証項目

- 無人の定期セッション（Routine/cron）で tl;dv / Notion コネクタが使えるか（使えれば定期自動化、不可なら半自動継続）。
- リマインド／通知の費用ゼロ実現方式。
- 手順の Claude スキル化（`.claude/skills/`）で「取り込んで」の一言運用。
