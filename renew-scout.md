# Renew Career スカウト一括送信スキル

Renew Career（renew-career.com）の保存した検索条件を使って、学生候補者に対してスカウトメッセージを一括送信する。完了後にSlackへ結果を通知する。

## 対象アカウント
ユーザーの指示に応じて対象アカウントを判別する:

| アカウント | 判別方法 | Slack Webhook 環境変数 |
|----------|---------|---------------|
| **GARO** | 右上に「株式会社GARO」と表示 | `$SLACK_WEBHOOK_GARO` |
| **Ship** | 右上に「株式会社ship」と表示 | `$SLACK_WEBHOOK_SHIP` |

- 「Shipのスカウト送って」→ Shipアカウントで実行、Ship用Slackに通知
- 「GAROのスカウト送って」または指定なし → GAROアカウントで実行、GARO用Slackに通知
- ページ表示後にスクリーンショットで右上のアカウント名を確認し、指定と異なる場合はユーザーにアカウント切り替えを依頼する

## 前提条件
- Chrome で Renew Career にログイン済みであること（https://renew-career.com/c/dashboard）
- Chrome MCP ツールが利用可能であること

## 実行フロー

### Phase 1: ブラウザ準備
1. `tabs_context_mcp` でタブ状況を確認
2. `tabs_create_mcp` で新しいタブを作成
3. `navigate` で絞り込み済みURLに直接遷移:
   ```
   https://renew-career.com/c/scouts?pref%5B0%5D=13&pref%5B1%5D=14&school_year%5B0%5D=1&school_year%5B1%5D=2&school_year%5B2%5D=7&graduation_year%5B0%5D=2027&graduation_year%5B1%5D=2028&graduation_year%5B2%5D=2029&graduation_year%5B3%5D=2030&graduation_year%5B4%5D=2031&graduation_year%5B5%5D=2032&occupation_type%5B0%5D=2&occupation_type%5B1%5D=1&occupation_type%5B2%5D=3&occupation_type%5B3%5D=6&occupation_type%5B4%5D=7&occupation_type%5B5%5D=8&occupation_type%5B6%5D=5&occupation_type%5B7%5D=11&industry%5B0%5D=1&industry%5B1%5D=3&industry%5B2%5D=4&hours_per_week=20
   ```
4. ページ読み込み待機（3秒）
5. スクリーンショットで右上のアカウント名がGAROであることを確認
6. スカウト可能数を確認（0なら終了）

### Phase 2: 候補者URL一覧を取得
`javascript_tool` で候補者の詳細ページURLを一括取得:
```javascript
const links = Array.from(document.querySelectorAll('a[href*="/c/scouts/detail/"]')).map(a => a.href);
const unique = [...new Set(links)];
JSON.stringify(unique);
```

### Phase 3: 各候補者にスカウト送信（JS一括実行方式）

取得したURLリストを上から順に処理する。各候補者に対して以下を実行:

#### Step 1: 候補者ページに遷移
- `navigate` で `https://renew-career.com/c/scouts/detail/{ID}` に移動
- ページ読み込み待機（2秒）

#### Step 2: JS一括送信（重要：この方法で実行すること）
`javascript_tool` で以下のコードを実行する。これにより「興味のある職種」判定→求人選択→テンプレートメッセージ設定→フォーム送信が1回で完了する:

```javascript
// 興味のある職種にエンジニアが含まれるか判定
const items = document.querySelectorAll('li');
let ji = '';
for (const li of items) {
  if (li.textContent.includes('興味のある職種')) {
    ji = li.textContent;
    break;
  }
}
const isEng = ji.includes('エンジニア');

// テンプレートメッセージ
const openMsg = 'はじめまして!\n\n株式会社GAROでインターンの採用を担当しております。\n大池と申します。\n\nプロフィールを拝見し、ぜひ一度お話しできればと思いご連絡いたしました!\n\nGAROは経営者・ビジネスパーソン向けSNS「Rep」を運営するスタートアップです。現在、事業推進や企画、マーケティングなど幅広い業務に携わっていただける総合職インターンを募集しています。\n\n「まだやりたいことが明確ではないけれど、成長環境に身を置きたい」という方にもぴったりです。様々な業務を経験しながら、自分の強みや志向性を見つけていける環境があります。\n\nご興味がありましたら、まずはカジュアルにお話しさせてください。\nご連絡をお待ちしております!';

const engMsg = 'はじめまして!\n\n株式会社GAROでインターンの採用を担当しております。\n大池と申します。\n\nプロフィールを拝見し、ぜひ一度お話しできればと思いご連絡いたしました!\n\nGAROは経営者・ビジネスパーソン向けSNS「Rep」を運営するスタートアップです。現在、プロダクト開発を一緒に進めてくれる学生エンジニアを募集しています。\n\nFlutterやNode.jsなどモダンな技術スタックで開発しており、実務経験を積みたいエンジニア志望の方にぴったりの環境です。\n\nご興味がありましたら、まずはカジュアルにお話しさせてください。\nご連絡をお待ちしております!';

// 求人チェックボックスをON + メッセージ設定 + フォーム送信
const form = document.querySelector('form[action="https://renew-career.com/c/scouts/send"]');
const msg = document.querySelector('#scout_message');
const coo = document.querySelector('#check-joboffer-1283');

if (form && coo) {
  coo.checked = true;
  msg.value = isEng ? engMsg : openMsg;
  form.submit();
  JSON.stringify({s:'ok', eng:isEng});
} else {
  JSON.stringify({s:'err'});
}
```

#### Step 3: 結果確認
- 送信後のレスポンスが `{"status":"success"}` であることを確認
- 2秒待機してから次の候補者へ

#### Step 4: 次の候補者へ
- URLリストの次の候補者に `navigate` で遷移
- Step 2〜3 を繰り返す
- スカウト可能数が0になったら即座にループを終了

### Phase 4: ページネーション
1ページ目の全候補者を処理したら、絞り込みURLに戻って次のページの候補者URLを取得し、同様に処理する。

### Phase 5: Slack通知
全候補者の処理完了後、Slack MCP（`mcp__slack__slack_post_message`）で `#garo-インターン採用`（チャンネルID: `C0A8N5UBPD1`）に通知:

```
📋 Renew Career スカウト送信完了

🕐 実行日時: YYYY/MM/DD HH:MM

合計: X名にスカウトを送信しました
残りスカウト数: Y/50
```

## 求人ID一覧（GARO）
| 求人 | チェックボックスID | value |
|------|-------------------|-------|
| COO候補募集（事業立ち上げ） | `#check-joboffer-1283` | 1283 |
| CMO候補募集（マーケティング） | `#check-joboffer-1433` | 1433 |
| CDO候補募集（デザイン） | `#check-joboffer-1434` | 1434 |
| CFO候補募集（財務） | `#check-joboffer-1435` | 1435 |

## エラーハンドリング
- JSの結果が `{"s":"err"}` の場合はスキップして次の候補者へ
- スカウト可能数が0になったらループを終了し、その旨をSlack通知に含める
- ページ遷移に失敗した場合は再試行（最大2回）

## 注意事項
- **UI操作（find→click）ではなく、必ずJS一括実行方式を使うこと**（速度と安定性のため）
- テンプレートメッセージはJSコード内に直接埋め込む（UIでテンプレート選択しない）
- 候補者が大量の場合、ページネーションに対応する
- スカウト残数を常に監視し、上限に達したら停止する
- 送信フォームのaction URL `https://renew-career.com/c/scouts/send` が変わっていないか確認する
