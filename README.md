# いずみんスキル

Claude Code のカスタムスキル集（採用自動化）

## スキル一覧

| スキル | 説明 |
|--------|------|
| `renew-scout` | Renew Career スカウト一括送信 |
| `renew-applicant-screen` | Renew Career 応募者スクリーニング＆面談お誘い |
| `lapras-interest` | LAPRAS Scout 興味通知一括送信 |
| `casual-interview-sync` | カジュアル面談 → Notion 自動同期 |

## セットアップ

### 1. スキルファイルを配置

```bash
# このリポジトリを ~/.claude/commands にクローン
git clone https://github.com/GARO-inc/izumin-skills.git ~/.claude/commands
```

### 2. 環境変数を設定

`~/.zshrc` または `~/.bashrc` に以下を追加:

```bash
export SLACK_WEBHOOK_GARO="https://hooks.slack.com/services/xxxxx"
export SLACK_WEBHOOK_SHIP="https://hooks.slack.com/services/xxxxx"
export SLACK_WEBHOOK_LAPRAS="https://hooks.slack.com/services/xxxxx"
```

実際のWebhook URLは社内のSlack管理者に確認してください。

### 3. 前提条件

- Chrome MCP 拡張機能がインストール済み
- 各サービス（Renew Career / LAPRAS Scout）にChromeでログイン済み
