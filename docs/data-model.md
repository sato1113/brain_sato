# データモデル（MASAKI OS）

Supabase（PostgreSQL）。全テーブルで RLS を有効化する。ここでは全体像と、初回縦串（slice-01）で実装するテーブルの DDL を示す。

## 1. テーブル一覧（全体像）

| 分類 | テーブル | 概要 | 初回縦串 |
|---|---|---|---|
| 基盤 | `projects` | 全データの紐付け先 | ✅ |
| 基盤 | `integrations` | 外部連携の設定・トークン参照 | ✅(最小) |
| 基盤 | `audit_logs` | 重要操作の監査記録 | ✅(最小) |
| タスク | `tasks` | タスク（会議由来の action_item を含む） | ✅ |
| タスク | `task_templates` | 定常タスクの雛形 | Phase2 |
| タスク | `task_runs` | 定常タスクの実行インスタンス | Phase2 |
| 会議 | `meetings` | 会議（tl;dv 由来） | ✅ |
| 会議 | `meeting_participants` | 参加者 | ✅ |
| 会議 | `meeting_minutes` | 生成した議事録 | ✅ |
| 会議 | `decisions` | 決定事項 | ✅ |
| 会議 | `confirmations` | 確認待ち事項 | ✅ |
| 成果物 | `artifacts` | Drive/Docs/GitHub 等の成果物リンク | ✅ |
| AI | `ai_work_logs` | AI作業履歴（要約・成果物中心） | Phase1後半 |
| 自動化 | `automations` | 自動処理の定義（3区分） | ✅(最小) |
| 自動化 | `automation_runs` | 実行ログ（成功/失敗/コスト） | ✅ |
| 自動化 | `reminders` | リマインド | Phase1後半 |
| KPI | `kpi_definitions` / `kpi_values` | KPI定義・値 | Phase2 |
| レポート | `reports` | 週次/月次レポート | Phase2 |

## 2. 共通方針

- 主キー：`id uuid default gen_random_uuid()`。
- 監査列：`created_at timestamptz default now()`、`updated_at`（トリガで更新）。
- 会議由来レコードには **`source_confidence`（ASR起因の要確認フラグ）** と **原文参照（tl;dvタイムスタンプURL）** を持たせる。
- ステータスで「下書き（draft）→確定（confirmed）」を管理し、確定まで外部連携を起こさない。
- 期限は「絶対日付（解決後）」と「原文表現（相対）」を両方保持する。

## 3. 初回縦串（slice-01）DDL

```sql
-- プロジェクト
create table projects (
  id            uuid primary key default gen_random_uuid(),
  name          text not null,
  description   text,
  status        text not null default 'active',   -- active | paused | done
  created_at    timestamptz not null default now(),
  updated_at    timestamptz not null default now()
);

-- 会議（tl;dv 由来）
create table meetings (
  id                uuid primary key default gen_random_uuid(),
  project_id        uuid references projects(id) on delete set null,
  tldv_id           text unique not null,          -- tl;dv meeting id（冪等キー）
  title             text not null,
  happened_at       timestamptz not null,
  duration_seconds  integer,
  organizer_email   text,
  meeting_url        text,                          -- tl;dv の URL
  transcript_status text not null default 'pending',-- pending | fetched | none
  minutes_status    text not null default 'pending',-- pending | generated | confirmed
  created_at        timestamptz not null default now(),
  updated_at        timestamptz not null default now()
);

-- 参加者
create table meeting_participants (
  id          uuid primary key default gen_random_uuid(),
  meeting_id  uuid not null references meetings(id) on delete cascade,
  name        text,
  email       text,
  role        text                                   -- organizer | invitee | speaker
);

-- 議事録（目標フォーマットの本文＋構造化）
create table meeting_minutes (
  id            uuid primary key default gen_random_uuid(),
  meeting_id    uuid not null references meetings(id) on delete cascade,
  body_markdown text not null,                        -- 目標フォーマットの議事録本文
  summary       text,                                 -- 2.結論の要約
  drive_file_id text,                                 -- Google Docs/Drive のファイルID
  drive_url     text,
  ai_model      text,                                 -- 生成に使ったモデル
  status        text not null default 'draft',        -- draft | confirmed
  created_at    timestamptz not null default now(),
  updated_at    timestamptz not null default now()
);

-- 決定事項
create table decisions (
  id            uuid primary key default gen_random_uuid(),
  meeting_id    uuid references meetings(id) on delete cascade,
  project_id    uuid references projects(id) on delete set null,
  body          text not null,
  needs_review  boolean not null default false,       -- ASR等による要原文確認
  source_ts_url text,                                  -- tl;dv 該当箇所リンク
  status        text not null default 'draft',         -- draft | confirmed
  created_at    timestamptz not null default now()
);

-- タスク（会議由来の action_item を含む横断テーブル）
create table tasks (
  id             uuid primary key default gen_random_uuid(),
  project_id     uuid references projects(id) on delete set null,
  meeting_id     uuid references meetings(id) on delete set null,
  decision_id    uuid references decisions(id) on delete set null, -- 決定との相互リンク
  title          text not null,
  assignee       text,                                 -- 自分/他者（社外含む）名 or email
  is_mine        boolean not null default false,       -- 自分のタスクか（依頼追跡用）
  due_date       date,                                 -- 解決済み絶対日付
  due_raw        text,                                 -- 原文の相対表現（「お盆明け」等）
  priority       text not null default 'normal',       -- low | normal | high
  status         text not null default 'draft',        -- draft | todo | in_progress | waiting_external | done
  external_ref   text,                                 -- WBS番号など既存管理表のID
  needs_review   boolean not null default false,
  source_ts_url  text,
  created_at     timestamptz not null default now(),
  updated_at     timestamptz not null default now()
);

-- 確認待ち事項
create table confirmations (
  id            uuid primary key default gen_random_uuid(),
  meeting_id    uuid references meetings(id) on delete cascade,
  project_id    uuid references projects(id) on delete set null,
  body          text not null,
  owner         text,                                  -- 誰の確認/回答待ちか
  status        text not null default 'open',          -- open | resolved
  needs_review  boolean not null default false,
  source_ts_url text,
  created_at    timestamptz not null default now()
);

-- 成果物（Drive/Docs/GitHub 等へのリンク）
create table artifacts (
  id           uuid primary key default gen_random_uuid(),
  project_id   uuid references projects(id) on delete set null,
  meeting_id   uuid references meetings(id) on delete set null,
  kind         text not null,                          -- minutes | slide | doc | recording | other
  title        text,
  url          text not null,
  storage      text,                                   -- google_drive | github | notion | other
  created_at   timestamptz not null default now()
);

-- 自動処理の実行ログ（成功/失敗/コスト）
create table automation_runs (
  id            uuid primary key default gen_random_uuid(),
  job           text not null,                          -- poll-meetings | generate-minutes | ...
  status        text not null,                          -- success | failed | partial
  started_at    timestamptz not null default now(),
  finished_at   timestamptz,
  items_processed integer default 0,
  error         text,
  cost_usd      numeric(10,4),                          -- AI/API 概算コスト
  meta          jsonb
);
```

`integrations`・`audit_logs`・`automations` は最小構成で用意し、詳細は各機能追加時に拡張する。

## 4. 主な関連（ER 概要）

```
projects 1─* meetings 1─* meeting_participants
                     1─1 meeting_minutes
                     1─* decisions 1─* tasks
                     1─* confirmations
                     1─* artifacts
projects 1─* tasks
projects 1─* artifacts
```
