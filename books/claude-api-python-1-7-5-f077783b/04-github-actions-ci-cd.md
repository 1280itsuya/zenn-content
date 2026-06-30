---
title: "GitHub Actions で毎朝自動投稿するCI/CDパイプラインの構築"
free: false
---

## GitHub Actions で毎朝自動投稿するCI/CDパイプラインの構築

### 全体の構成

この章では、Python スクリプトで生成した記事を Zenn・Qiita へ自動投稿するパイプラインを GitHub Actions の cron ジョブとして組む手順を解説する。ローカルで手動実行していた処理をそのままクラウドに移すだけなので、既存コードの変更はほぼ不要だ。

```
ローカル生成スクリプト
      ↓ git push
GitHub リポジトリ
      ↓ cron trigger (毎朝 7:00 JST)
GitHub Actions Runner
      ↓ Python スクリプト実行
Zenn (git push) / Qiita (REST API)
```

---

### PAT の安全な管理

最初に躓くのが認証情報の扱いだ。**絶対に `.env` ファイルをリポジトリに含めてはいけない**。代わりに GitHub Secrets を使う。

1. リポジトリ → **Settings → Secrets and variables → Actions** を開く
2. 「New repository secret」で以下を登録する

| シークレット名 | 値の例 |
|---|---|
| `QIITA_TOKEN` | Qiita の個人アクセストークン |
| `ZENN_GH_PAT` | Zenn 連携用 GitHub PAT（`repo` スコープのみ） |
| `OPENAI_API_KEY` | 記事生成に使う OpenAI キー |

PAT を発行するときは **スコープを最小限に絞る**こと。Zenn 連携なら `repo` だけで十分で、`admin` や `delete_repo` は不要だ。有効期限も 90 日に設定し、ローテーション運用を習慣づける。

---

### ワークフロー YAML の実装

`.github/workflows/daily_post.yml` を以下の内容で作成する。

```yaml
name: Daily Auto Post

on:
  schedule:
    # 毎朝 07:00 JST = 22:00 UTC (前日)
    - cron: "0 22 * * *"
  workflow_dispatch:  # 手動実行ボタンも残す

jobs:
  post:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.ZENN_GH_PAT }}

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: "pip"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Generate and post articles
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          QIITA_TOKEN: ${{ secrets.QIITA_TOKEN }}
          GIT_TERMINAL_PROMPT: "0"  # 認証ダイアログを出さない
        run: python src/orchestrator.py

      - name: Push Zenn articles
        run: |
          git config user.email "bot@example.com"
          git config user.name "AutoPostBot"
          git add articles/
          git diff --cached --quiet || git commit -m "auto: $(date '+%Y-%m-%d') 記事追加"
          git push https://x-access-token:${{ secrets.ZENN_GH_PAT }}@github.com/${{ github.repository }}.git HEAD:main
```

ポイントは `git push` の URL に PAT を直接埋め込む書き方だ。`GIT_TERMINAL_PROMPT: "0"` を忘れると、TTY がない Runner 上で認証ダイアログ待ちになり、ジョブが 6 時間後にタイムアウトする。これは実際にハマりやすい落とし穴だ。

---

### よくある失敗と対処

**① cron が動かない**
GitHub Actions の cron は UTC で指定する。日本時間 7:00 は `0 22 * * *`（前日 22:00 UTC）。ズレると意図しない時刻に実行される。`workflow_dispatch` を必ずセットで書いておくと、手動で即時テストできて便利だ。

**② Qiita が 429 で弾かれる**
Qiita の API レート制限は 1 時間あたり 1,000 リクエストだが、複数ジョブが同時に走ると超過しやすい。スクリプト側で投稿間に `time.sleep(5)` を挟み、最低でも 5 時間以上の間隔を `QIITA_MIN_INTERVAL_HOURS` 環境変数で制御する設計にしておくこと。

**③ Secrets が `None` になる**
フォークされたリポジトリや Pull Request トリガーでは Secrets が渡らない仕様になっている。`on: schedule` と `workflow_dispatch` だけをトリガーにすれば問題ない。

---

### 動作確認の手順

1. ワークフローファイルを `main` ブランチに push する
2. Actions タブ → **Run workflow** で手動実行する
3. ログの `Generate and post articles` ステップで記事数と投稿先を確認する
4. Qiita の「マイページ」と Zenn のダッシュボードで記事が反映されていれば成功だ

初回は必ず手動実行で通しテストをしてから cron に任せること。cron で最初に動いたときにエラーが出ると、翌朝まで気づかないまま丸一日ロスする。
