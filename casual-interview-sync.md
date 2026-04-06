# カジュアル面談 → Notion 自動同期スキル

Google Calendar の今日のカジュアル面談予定を検出し、Notion の候補者タレントプール（選考管理DB）に自動追加する。既にNotionに存在する候補者はスキップする。

## 前提条件
- Google Calendar MCP が利用可能であること
- Notion MCP が利用可能であること

## Notion 候補者タレントプール
- データベースID: `3378f60eee2381c99716e0f87df12d52`
- データソースURL: `collection://3378f60e-ee23-811c-ba5d-000b2749adcc`
- URL: https://www.notion.so/garoinc/3378f60eee2381c99716e0f87df12d52

### プロパティ
| プロパティ | 型 | 用途 |
|-----------|------|------|
| 候補者名 | title | 候補者のフルネーム |
| 選考ステータス | select | 「カジュアル面談予定」を設定 |
| 次アクション日 | text | 面談日（YYYY-MM-DD） |
| 媒体 | select | Renew 等 |
| 職種 | select | エンジニア / デザイナー / PM 等 |
| カジュアル面談メモ | text | メモ欄 |
| プロフィールURL | url | プロフィールリンク |
| GARO点数 | number | 評価スコア |
| Ship点数 | number | 評価スコア |

## カレンダーイベントの形式
タイトル: `{候補者名} x {面談者名}: カジュアル面談`

例: `加唐創 x 大池泉美: カジュアル面談`

- 「x」の前の部分 = 候補者名（前後の空白をtrim）
- attendees のうち `@garoinc.jp` でないメールアドレス = 候補者のメール

## 実行フロー

### Step 1: 今日の日付を取得
Google Calendar MCP の `get-current-time` で現在時刻を取得し、今日の日付範囲を決定する。
- timeMin: 今日の 00:00:00 (Asia/Tokyo)
- timeMax: 今日の 23:59:59 (Asia/Tokyo)

### Step 2: Google Calendar からイベント取得
`list-events` で今日のイベントを取得する。
- calendarId: "primary"
- timeZone: "Asia/Tokyo"

### Step 3: カジュアル面談イベントをフィルタ
`summary` に「カジュアル面談」を含むイベントだけ抽出する。

各イベントから以下を抽出:
- **候補者名**: タイトルの「x」の前の部分（trimする）
- **候補者メール**: attendees の中で `@garoinc.jp` でないアドレス
- **面談日時**: start.dateTime
- **Google Meet リンク**: hangoutLink または conferenceData
- **カレンダーリンク**: htmlLink

面談が0件なら「今日のカジュアル面談はありません」と報告して終了。

### Step 4: Notion で重複チェック
Notion の候補者タレントプールを検索（data_source_url: `collection://3378f60e-ee23-811c-ba5d-000b2749adcc`）し、同名の候補者が既に存在するか確認する。

### Step 5: Notion に候補者を追加
重複がなければ Notion に新規ページを作成する。

- parent: `{ "type": "data_source_id", "data_source_id": "3378f60e-ee23-811c-ba5d-000b2749adcc" }`
- properties:
  - 候補者名: 抽出した候補者名
  - 選考ステータス: "カジュアル面談予定"
  - 次アクション日: 面談日（YYYY-MM-DD形式）
- content（ページ本文）:
  ```
  ## カジュアル面談情報

  - **日時**: YYYY-MM-DD HH:MM〜HH:MM
  - **メール**: candidate@example.com
  - **Google Meet**: https://meet.google.com/xxx
  - **カレンダー**: https://www.google.com/calendar/event?eid=xxx
  ```

### Step 6: 結果報告
処理結果をまとめて報告する。

```
📋 カジュアル面談 → Notion 同期結果（YYYY-MM-DD）

✅ 追加: 山田太郎（13:00〜13:30）
✅ 追加: 鈴木花子（15:00〜15:30）
⏭️ スキップ（既存）: 佐藤一郎

合計: 追加 2名 / スキップ 1名
```
