# カジュアル面談 → Notion 自動同期スキル

Google Calendar（ブラウザ）から今日のカジュアル面談予定を検出し、Notion の候補者タレントプール（選考管理DB）に自動追加する。既にNotionに存在する候補者はスキップする。

## 前提条件
- Chrome で Google Calendar にログイン済みであること（@garoinc.jp アカウント）
- Chrome MCP ツールが利用可能であること
- Notion MCP が利用可能であること

## Notion 候補者タレントプール
- データソースURL: `collection://3378f60e-ee23-811c-ba5d-000b2749adcc`
- URL: https://www.notion.so/garoinc/3378f60eee2381c99716e0f87df12d52

### プロパティ
| プロパティ | 型 | 設定値 |
|-----------|------|------|
| 候補者名 | title | 候補者のフルネーム |
| 選考ステータス | select | 「カジュアル面談予定」 |
| 次アクション日 | text | 面談日（YYYY-MM-DD） |

## カレンダーイベントの形式
タイトル: `{候補者名} x {面談者名}: カジュアル面談`

例: `加唐創 x 大池泉美: カジュアル面談`

- 「x」の前の部分 = 候補者名（前後の空白をtrim）
- 「:」の後に「カジュアル面談」を含むものが対象

## 実行フロー

### Phase 1: ブラウザ準備
1. `tabs_context_mcp` でタブ状況を確認
2. `tabs_create_mcp` で新しいタブを作成
3. `navigate` で `https://calendar.google.com/calendar/r/day` に移動（今日の日表示）
4. ページ読み込み待機（3秒）

### Phase 2: 今日のイベントを取得
1. スクリーンショットで今日のカレンダー表示を確認
2. `get_page_text` または `javascript_tool` でページ内のイベント情報を取得
3. 「カジュアル面談」を含むイベントを特定

イベントが見つからなかった場合の代替手段:
- `javascript_tool` で DOM からイベント要素を抽出:
  ```javascript
  // イベント要素からテキストを取得
  const events = document.querySelectorAll('[data-eventid]');
  const results = [];
  events.forEach(e => {
    const text = e.textContent || e.innerText;
    if (text.includes('カジュアル面談')) {
      results.push(text);
    }
  });
  return JSON.stringify(results);
  ```

### Phase 3: 各イベントの詳細を取得
「カジュアル面談」イベントごとに:

1. イベントをクリックして詳細ポップアップを開く
2. スクリーンショットで詳細情報を確認
3. 以下を読み取る:
   - **候補者名**: タイトルの「x」の前の部分
   - **面談日時**: 日付と時刻
   - **候補者メール**: ゲスト一覧から `@garoinc.jp` でないアドレス
   - **Google Meet リンク**: 会議リンク
4. ポップアップを閉じる（×ボタンまたはEscキー）

### Phase 4: Notion で重複チェック
Notion MCP の検索ツールで候補者タレントプール（data_source_url: `collection://3378f60e-ee23-811c-ba5d-000b2749adcc`）を検索し、同名の候補者が既に存在するか確認する。

### Phase 5: Notion に候補者を追加
重複がなければ Notion MCP のページ作成ツールで追加する。

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
  ```

### Phase 6: 結果報告
処理結果をまとめて報告する。

```
📋 カジュアル面談 → Notion 同期結果（YYYY-MM-DD）

✅ 追加: 山田太郎（13:00〜13:30）
✅ 追加: 鈴木花子（15:00〜15:30）
⏭️ スキップ（既存）: 佐藤一郎

合計: 追加 2名 / スキップ 1名
```

面談が0件の場合: 「今日のカジュアル面談はありません」と報告して終了。

## エラーハンドリング
- カレンダーページが読み込めない場合はリトライ（最大2回）
- イベント詳細が読み取れない場合はスクリーンショットで確認して再試行
- Notion への追加が失敗した場合はスキップしてログに残す

## 注意事項
- Google Calendar は `@garoinc.jp` アカウントでログインしていること
- 「カジュアル面談」を含まないイベントはスキップ
- 候補者名はタイトルの「x」の前の部分を trim して使用
