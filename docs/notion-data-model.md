# Notion データベース設計（MASAKI OS）

Notion をデータ基盤兼ダッシュボードとして使う。以下のデータベースを MASAKI OS 用スペース（またはページ）配下に作成する。プロパティ名は日本語表示・英語キーの対応を併記。

## 0. 共通方針

- 各DBは相互に **リレーション** で結ぶ（プロジェクトをハブに）。
- 会議由来レコードは **要確認（needs_review）** と **原文リンク（tl;dv タイムスタンプ）** を持つ（ASR誤り対策）。
- ステータスで **下書き（draft）→確定（confirmed）** を管理し、確定まで外部連携しない。
- 期限は **絶対日付** と **原文表現（相対）** を両方保持。

## 1. データベース一覧

| DB | 用途 | Phase |
|---|---|---|
| Projects | 全データの紐付け先 | 1 |
| Meetings | 会議（tl;dv 由来）＋議事録本文 | 1 |
| Decisions | 決定事項 | 1 |
| Tasks | タスク（会議由来の action item を含む） | 1 |
| Confirmations | 確認待ち事項 | 1 |
| Artifacts | 成果物リンク（Drive/Docs/GitHub 等） | 1（任意） |
| Automation Runs | 取り込み等の実行ログ | 1 |
| AI Work Logs | AI作業履歴（要約・成果物中心） | 2 |

## 2. 各DBのプロパティ

### Projects
| プロパティ | 型 | 備考 |
|---|---|---|
| 名前 (name) | Title | |
| 説明 (description) | Text | |
| ステータス (status) | Select | active / paused / done |

### Meetings
| プロパティ | 型 | 備考 |
|---|---|---|
| タイトル (title) | Title | |
| プロジェクト (project) | Relation → Projects | |
| tl;dv ID (tldv_id) | Text | 冪等キー（一意運用） |
| 開催日時 (happened_at) | Date | |
| 時間(分) (duration_min) | Number | |
| 主催者 (organizer) | Text | |
| tl;dv URL (meeting_url) | URL | |
| 議事録状態 (minutes_status) | Select | pending / generated / confirmed |
| （本文） | Page body | 目標フォーマットの議事録を本文に記載 |

### Decisions
| プロパティ | 型 | 備考 |
|---|---|---|
| 内容 (body) | Title | |
| コード (code) | Text | D1, D2 … |
| 会議 (meeting) | Relation → Meetings | |
| プロジェクト (project) | Relation → Projects | |
| 要確認 (needs_review) | Checkbox | ASR等 |
| 原文リンク (source_ts_url) | URL | tl;dv 該当箇所 |
| ステータス (status) | Select | draft / confirmed |

### Tasks
| プロパティ | 型 | 備考 |
|---|---|---|
| タイトル (title) | Title | |
| プロジェクト (project) | Relation → Projects | |
| 会議 (meeting) | Relation → Meetings | |
| 関連決定 (decision) | Relation → Decisions | |
| 担当 (assignee) | Text | 自分/他者（社外含む） |
| 自分のタスク (is_mine) | Checkbox | 依頼追跡用 |
| 期限 (due_date) | Date | 解決済み絶対日付 |
| 期限(原文) (due_raw) | Text | 「お盆明け」等 |
| 優先度 (priority) | Select | low / normal / high |
| ステータス (status) | Select | draft / todo / in_progress / waiting_external / done |
| 外部ID (external_ref) | Text | WBS番号など |
| 要確認 (needs_review) | Checkbox | |
| 原文リンク (source_ts_url) | URL | |

### Confirmations
| プロパティ | 型 | 備考 |
|---|---|---|
| 内容 (body) | Title | |
| 会議 (meeting) | Relation → Meetings | |
| プロジェクト (project) | Relation → Projects | |
| 待ち先 (owner) | Text | 誰の確認/回答待ちか |
| ステータス (status) | Select | open / resolved |
| 要確認 (needs_review) | Checkbox | |
| 原文リンク (source_ts_url) | URL | |

### Artifacts（任意）
| プロパティ | 型 | 備考 |
|---|---|---|
| タイトル (title) | Title | |
| プロジェクト (project) | Relation → Projects | |
| 会議 (meeting) | Relation → Meetings | |
| 種別 (kind) | Select | minutes / slide / doc / recording / other |
| URL (url) | URL | |
| 保存先 (storage) | Select | google_drive / notion / github / other |

### Automation Runs
| プロパティ | 型 | 備考 |
|---|---|---|
| ジョブ (job) | Title | poll-meetings / generate-minutes 等 |
| 実行日時 (ran_at) | Date | |
| 結果 (status) | Select | success / failed / partial |
| 処理件数 (items) | Number | |
| エラー (error) | Text | |

## 3. ダッシュボード（Notion ビュー）

MASAKI OS ホームページに、以下のリンクドビューを並べる。

- **今日/今週のタスク**：Tasks を due_date でフィルタ（is_mine=✓ を上に）
- **期限超過**：Tasks status≠done かつ due_date<今日
- **確認待ち**：Confirmations status=open ＋ 要確認(needs_review)=✓ の Tasks/Decisions
- **直近の会議**：Meetings を happened_at 降順
- **プロジェクト別**：Tasks を project でグループ化

## 4. 関連（リレーション概要）

```
Projects ─< Meetings ─< Decisions ─< Tasks
                     ─< Confirmations
                     ─< Artifacts
Projects ─< Tasks
Projects ─< Artifacts
```
