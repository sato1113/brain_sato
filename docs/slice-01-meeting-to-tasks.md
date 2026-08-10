# 初回縦串 仕様：会議 → 議事録 → タスク/決定事項

MVPの最優先。Claude が tl;dv コネクタから会議を取り込み、議事録を生成して Notion に登録し、決定事項・タスク・確認待ちを抽出して Notion に登録、Notion のビューで確認・確定できるまでを1本通す。**自前アプリ・APIキー・課金なし**（Claude がコネクタを操作して実行）。

## 1. 処理フロー

```
[1] Claude：tl;dv search-meetings で新規会議を検出（tldv_id で冪等）
      └─ Notion「Meetings」に登録／更新
[2] Claude：tl;dv get-meeting-transcript で全文取得（無ければ minutes_status=none で停止）
[3] Claude：文字起こし → 議事録本文（目標フォーマット, §3）を生成
      └─ Notion「Meetings」ページ本文に記載（minutes_status=generated）
      └─（任意）Google Drive にも保存し Artifacts にリンク登録
[4] Claude：議事録＋文字起こしから構造化抽出（§4 スキーマ）
      └─ Notion「Decisions / Tasks / Confirmations」に status=draft で登録
[5] 自分：Notion で内容を確認 → status=confirmed に変更
      └─ 確定後のみ、依頼送付・リマインド等の連携を有効化（Phase2）
```

- 各実行は Notion「Automation Runs」に記録（ジョブ名・件数・エラー）。
- 冪等性：`Meetings.tldv_id` を一意運用し、再実行で重複登録しない。既存会議は差分のみ更新。
- 実行主体は Claude セッション（Phase1は対話的に起動）。

## 2. スコープ（この縦串でやること / やらないこと）

**やる**：tl;dv取り込み、議事録生成、Notion登録（＋任意でDrive保存）、決定/タスク/確認待ちの抽出と登録、Notionでの確認・確定、実行ログ。

**やらない（後続）**：リマインド送信、他者への依頼の自動送付、定期自動実行（Phase2で可否検証）、KPI集計、プロジェクトの自動判定（当面は手動 or 単純ルール）。

## 3. 議事録フォーマット（確定）

```
会議名：
開催日時：YYYY-MM-DD（曜）HH:MM–HH:MM JST（N分）
参加者：
関連プロジェクト：

1. 会議の目的
2. 結論
3. 決定事項          … D1, D2, … と採番
4. 議論内容（要点）
5. タスク            … 構造化抽出（§4）と対応。担当・期限つき
6. 確認待ち事項
7. 次回までに行うこと
8. 関連資料
```

- 明らかなASR誤認識は整形時に補正するが、**数値・金額・氏名は「要原文確認」を残す**。
- 決定事項は採番（D1…）し、対応するタスクと相互参照する。

## 4. 構造化抽出スキーマ（確定）

AI の出力を次の JSON に固定し、DB へマッピングする。

```json
{
  "decisions": [
    { "code": "D1", "body": "…", "project_hint": "…",
      "needs_review": false, "source_ts_url": "https://tldv.io/app/meetings/…?t=…" }
  ],
  "tasks": [
    { "title": "…", "assignee": "…", "is_mine": false,
      "due_raw": "お盆明け", "due_date": "2026-08-20",
      "priority": "normal", "status": "draft",
      "related_decision_code": "D4", "external_ref": "WBS114",
      "needs_review": false, "source_ts_url": "…" }
  ],
  "confirmations": [
    { "body": "…", "owner": "…", "needs_review": false, "source_ts_url": "…" }
  ]
}
```

**タスク抽出の粒度（確定）**：`title` / `assignee` / `due（raw+解決日）` / `related_decision` の4点セットを基本とし、`is_mine`（自分の担当か＝依頼追跡）と `external_ref`（WBS番号等）を任意で付与する。抽出JSONは Notion の各DBプロパティ（notion-data-model.md §2）へマッピングする。

### 期限の相対表現の解決
会議日（`Meetings.happened_at`）を基準に、「本日中」「明日」「今週金曜」「お盆明け(=8/19–20目安)」等を絶対日付へ変換。曖昧なものは `due_date` を空にし `due_raw` のみ残す。

### ASR対策
`needs_review=true` の項目は Notion ビューで強調（フィルタ）し、`source_ts_url`（tl;dv の該当箇所）から原文へ即ジャンプできるようにする。

## 5. 自動化区分（この縦串）

| ステップ | 区分 | 理由 |
|---|---|---|
| 会議取り込み・議事録生成・（任意でDrive保存）・抽出 | 自動（auto） | 下書き生成まで。副作用は自分の保存領域のみ |
| 議事録・決定・タスクの「確定」 | 承認後（approval_required） | 自分の確認で draft→confirmed |
| 他者への依頼送付・リマインド | 承認後（Phase2） | 確定後のみ有効化 |
| 会員情報変更・報酬確定・個人情報削除 | 手動（manual_only） | 本縦串では扱わない |

## 6. 受け入れ条件（Definition of Done）

- [ ] tl;dv の指定会議を取り込み、Notion「Meetings」に登録される（再実行で重複しない）。
- [ ] 文字起こしから §3 フォーマットの議事録が生成され、Meetings ページ本文に記載される（minutes_status=generated）。
- [ ] 決定事項・タスク・確認待ちが §4 スキーマで抽出され、Notion の各DBに status=draft で登録される。
- [ ] タスクに担当・期限（raw＋解決日）・関連決定が入る。相対期限が絶対日付に解決される。
- [ ] `needs_review` 項目が Notion ビューで抽出でき、`source_ts_url` から原文へ飛べる。
- [ ] Notion のビューで一覧・確認し、status を `confirmed` にできる。
- [ ] 実行が Notion「Automation Runs」に記録され、失敗時にエラーが残る。

## 7. 検証用リファレンス

実データ（システム会社MTG, 2026-08-06）で生成した議事録・抽出結果を
[docs/samples/system-mtg-2026-08-06.md](samples/system-mtg-2026-08-06.md) に置く。
実装時はこのサンプルを期待出力の基準（ゴールデンサンプル）として用いる。

## 8. 前提

追加の資格情報・課金は不要。必要なのは **接続済みコネクタ（tl;dv / Notion、任意で Google Drive）** のみ。
実行は Claude セッションが担う（議事録生成・抽出は Claude 自身が行う）。将来この手順を Claude スキル化して「取り込んで」で再現可能にする。
