# LAPRAS Scout 興味通知一括送信スキル

LAPRAS Scoutの保存した検索条件を使って、タレントプールに未登録の候補者に対して興味通知を一括送信する。完了後にSlackへ結果を通知する。

## 前提条件
- Chrome で LAPRAS Scout にログイン済みであること
- Chrome MCP ツールが利用可能であること

## 実行フロー

### Phase 1: ブラウザ準備
1. `tabs_context_mcp` でタブ状況を確認
2. `tabs_create_mcp` で新しいタブを作成
3. `navigate` で https://scout.lapras.com/lapras_search に移動

### Phase 2: 保存した条件ごとにループ（3条件）

対象条件（上から順に）:
- **PM**: 【PM】プロジェクトマネージャー領域
- **フロントエンド**: 【Eng】フロントエンド／AIエージェント
- **バックエンド**: 【Eng】バックエンド

各条件で以下を実行:

#### Step 1: 条件ロード
- `find` で「保存した条件」ボタンを見つけてクリック
- 対象条件をクリック

#### Step 2: フィルター調整
- JSで「タレントプールにいる候補者」のチェックボックスを外す:
```javascript
const span = [...document.querySelectorAll('span')].find(el => el.textContent.trim() === 'タレントプールにいる候補者');
const container = span.parentElement;
const checkbox = container.querySelector('input[type="checkbox"]') || container.parentElement.querySelector('input[type="checkbox"]');
if (checkbox) checkbox.click();
```
- `find` で検索ボタンを見つけてクリック

#### Step 3: 候補者数確認
- スクリーンショットで「該当する候補者: N名」を確認
- 0名ならこの条件をスキップして次へ

#### Step 4: 各候補者を処理
候補者リストを上から順に:

1. **名前リンクをクリック**（`find` でref取得→click）→ 新タブで開く
2. **プロフィール確認**: 右サイドバーから雇用形態を読み取る
   - 「フリーランス」→ 副業求人
   - 「正社員」「社員」→ 社員求人
3. **「タレントプールに追加」ボタン**をクリック
4. **ダイアログ設定**:
   - `form_input` で「追加するリストを選択」→ value=`31930`（興味リアクション待ち）
   - `form_input` で「追加時に興味通知を送信」→ `true`
   - `form_input` で「該当の求人」→ 条件と雇用形態に応じて:

| 条件 | 社員 | 副業/フリーランス |
|------|------|-------------------|
| PM | 10160 | 10160 |
| フロントエンド | 9528 | 11562 |
| バックエンド | 11566 | 10145 |

5. **「追加する」ボタン**をクリック（`find` でref取得→click）
6. 完了確認（ダイアログが閉じたかスクショで確認）
7. 候補者名・条件・求人をリストに記録
8. タブを閉じて検索結果タブに戻る
9. 次の候補者へ（ページ内の次の候補者、なければ次ページ）

### Phase 3: Slack通知
全条件の処理完了後、以下の形式でSlackに通知:

```
📋 LAPRAS Scout 興味通知送信完了

🕐 実行日時: YYYY/MM/DD HH:MM

【PM】N名
- 候補者A（社員/PM求人）
- 候補者B（社員/PM求人）

【フロントエンド】N名
- 候補者C（副業/FE求人）
- 候補者D（社員/FE求人）

【バックエンド】N名
- 候補者E（社員/BE求人）

合計: X名に興味通知を送信しました
```

Slack通知は Bash ツールで curl を使って環境変数の Webhook URL にPOSTする:
```bash
curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"text": "上記メッセージ"}' \
  "$SLACK_WEBHOOK_LAPRAS"
```

## 注意事項
- 候補者が大量の場合、ページネーションに対応する（ページ下部の次ページリンク）
- エラーが発生した場合はスキップしてログに残し、次の候補者へ進む
- 各候補者の処理後にスクショで成功確認を行う（2-3人に1回）
