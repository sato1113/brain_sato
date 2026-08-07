# サンプル：システム会社MTG（2026-08-06）

初回縦串の期待出力（ゴールデンサンプル）。tl;dv の実データ（meeting id: `6a7414bf8022840013ed5948`）の文字起こしから生成。
※ 元データはASR由来のため、数値・金額・氏名には「要原文確認」を含む。

## 生成された議事録（目標フォーマット）

```
会議名：システム会社MTG（報酬計算システム定例）
開催日時：2026-08-06（木）14:00–14:41 JST（41分）
参加者：田口雅哉（進行/運用）、山縣智史（開発）、武政寛興、kamio（satoshi-vision）、佐藤将貴（主催）
関連プロジェクト：報酬計算システム改修 / 認定代理店（認定パートナー）

1. 会議の目的
   WBS進捗レビューと、7月実績の仮計算（金曜必須）に向けた明細仕様・昇格判定・個別対応の確定。

2. 結論
   ・7月仮計算を今日〜明日で実施。インボイス調整額の明細レイアウトを確定して込みで公開。
   ・月額7ヶ月目の報酬カットは、今回はハンド対応＋ロジック追加の二段構えで対応。

3. 決定事項
   D1. 昇格判定は6月計算分から「緩和条件」で行う（現状は緩和条件のみ存在）。
   D2. ダッシュボード／組織図（釣り・角タイプ）は8/19（お盆明け）公開。
   D3. WBS No.113・114・117・118 はクローズ。No.114 は本番適用（本日更新）。
   D4. インボイスPDF明細：源泉所得税を「適用率」の右へ配置。控除枠を1段増やし ①②③ の番号と計算式（①+②−③）を記載、各合計＋振込額を表示。
   D5. 新規会員登録の確認画面を2回→1回（最終確認のみ）に統一。修正ボタンで修正画面へ戻す。登録後の変更は公式LINE/問い合わせ＋本人確認で対応する運用は継続。
   D6. バックオフィスに「未来テラシー」（アルファベット社提携商品）への遷移ボタンを追加。

4. 議論内容（要点）
   ・昇格エビデンス：どちらの基準で昇格したかを示せるよう、GMのグループボリューム量と40%ルール該当を可視化。
   ・インボイス調整額のマイナス表記は付けず、控除枠＋計算式表示で分かりやすくする方針。
   ・過払いコミッションの消し込み（1〜3月再検査分）が未完のため、代理店リスト＋調整金削除依頼を先行。

5. タスク（下記「構造化抽出」参照）

6. 確認待ち事項
   ・会員IDをクエリパラメータで保持したまま遷移できるか（要検証。別ボタンで実績あり）。
   ・未来テラシー遷移ボタンの送信データ項目名・URL・アイコン（アルファベット社へ確認中）。

7. 次回までに行うこと
   ・7月実績仮計算の公開、月額7ヶ月目対象者リストの共有、明細計算式の反映。

8. 関連資料
   ・Slack（計算エビデンス貼付）、WBS、ステージング環境、SV5 CSVファイル。
```

## 構造化抽出（DB登録形）

```json
{
  "decisions": [
    { "code": "D1", "body": "昇格判定は6月計算分から緩和条件で行う（現状は緩和条件のみ）", "needs_review": false },
    { "code": "D2", "body": "ダッシュボード/組織図は8/19（お盆明け）公開", "needs_review": false },
    { "code": "D3", "body": "WBS No.113/114/117/118 はクローズ。No.114 は本番適用（本日更新）", "needs_review": true },
    { "code": "D4", "body": "インボイスPDF明細：源泉所得税を適用率の右へ、控除枠①②③＋計算式①+②−③、各合計＋振込額を表示", "needs_review": true },
    { "code": "D5", "body": "新規会員登録の確認画面を2回→1回（最終確認のみ）に統一", "needs_review": false },
    { "code": "D6", "body": "バックオフィスに未来テラシー遷移ボタンを追加", "needs_review": false }
  ],
  "tasks": [
    { "title": "No.114 を本番適用・本日更新", "assignee": "山縣", "is_mine": false, "due_raw": "本日", "due_date": "2026-08-06", "related_decision_code": "D3", "external_ref": "WBS114", "needs_review": true },
    { "title": "インボイスPDF明細レイアウト修正（源泉を適用率の右、控除枠①②③＋計算式、各合計＋振込額）。Slackで案提示", "assignee": "山縣", "is_mine": false, "due_raw": "仮計算まで", "due_date": "2026-08-07", "related_decision_code": "D4", "needs_review": false },
    { "title": "明細の番号設定＋計算式記載処理の追加・公開", "assignee": "山縣", "is_mine": false, "due_raw": "明日まで", "due_date": "2026-08-07", "related_decision_code": "D4", "needs_review": false },
    { "title": "旧システム「コミュニケーション表示」のエラー確認（池田裕子データ）", "assignee": "山縣", "is_mine": false, "due_raw": "直近", "due_date": null, "needs_review": true },
    { "title": "月額7ヶ月目の報酬カット判定ロジックを追加", "assignee": "山縣", "is_mine": false, "due_raw": null, "due_date": null, "needs_review": false },
    { "title": "今回分の7ヶ月目対象者リストをハンド作成・本日中共有", "assignee": "田口", "is_mine": false, "due_raw": "本日中", "due_date": "2026-08-06", "needs_review": false },
    { "title": "先週定例の議事録から7ヶ月目カット要件を洗い出し共有", "assignee": "田口", "is_mine": false, "due_raw": null, "due_date": null, "needs_review": false },
    { "title": "未来テラシー遷移ボタン追加（URL遷移＋会員IDクエリ保持の検証）", "assignee": "山縣", "is_mine": false, "due_raw": "お盆明け", "due_date": "2026-08-20", "related_decision_code": "D6", "needs_review": false },
    { "title": "未来テラシーのアイコン・URL・項目名をアルファベット社に確認しSTWへ共有", "assignee": "田口", "is_mine": false, "due_raw": null, "due_date": null, "related_decision_code": "D6", "needs_review": false },
    { "title": "新規会員登録 確認画面を1回に修正", "assignee": "山縣", "is_mine": false, "due_raw": "8/19目安", "due_date": "2026-08-19", "related_decision_code": "D5", "needs_review": false },
    { "title": "過払い消し込み（代理店リスト＋調整金削除依頼）、7月度計算依頼の展開", "assignee": "山縣", "is_mine": false, "due_raw": "本日中着手", "due_date": "2026-08-06", "needs_review": false },
    { "title": "昇格判定を月初に実行できるようにする", "assignee": "山縣", "is_mine": false, "due_raw": "来月以降", "due_date": null, "related_decision_code": "D1", "needs_review": false }
  ],
  "confirmations": [
    { "body": "会員IDをクエリパラメータで保持したまま画面遷移できるか（要検証）", "owner": "山縣", "needs_review": false },
    { "body": "未来テラシー遷移ボタンの送信データ項目名・URL・アイコン", "owner": "田口→アルファベット社", "needs_review": false }
  ]
}
```

## メモ（この会議の主催者=佐藤将貴の観点）

- 自分（主催）の直接タスクはこの回では少なく、**社外（山縣/STW）への依頼と田口の対応待ち**が中心。→ 「他者への依頼追跡」「確認待ち」の可視化価値が高いことを裏付ける実例。
- `needs_review=true` が付くのは主に WBS番号・氏名・金額。ダッシュボードで原文（tl;dv）への即ジャンプが要る。
