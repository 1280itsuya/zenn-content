---
title: "GitHub Actionsでパイプラインを毎朝自動実行するCI設定"
free: false
---

## GitHub Actionsでパイプラインを毎朝自動実行するCI設定

ローカルで動くスクリプトができたら、次は「人間が何もしなくても毎朝動く」状態にする。GitHub Actionsのcronトリガーを使えば、サーバー不要・無料枠内で自動実行が完結する。

---

## cronトリガーの設定

GitHub ActionsのスケジュールはUTC基準で動く。日本時間の**毎朝7時に実行**したい場合は、UTC-9h = 22時を指定する。

```yaml
on:
  schedule:
    - cron: '0 22 * * *'   # UTC 22:00 = JST 07:00
  workflow_dispatch:         # 手動実行ボタンも残しておく
```

`workflow_dispatch` を併記しておくと、デバッグ時にActions画面から即実行できて便利だ。cronだけだと失敗の原因切り分けが難しいので必ず入れること。

---

## APIキーはSecretsに格納する

`.env` ファイルをリポジトリに入れてはいけない。GitHub Secretsに登録し、ワークフロー内で環境変数として注入する。

**登録手順：**
1. リポジトリ → Settings → Secrets and variables → Actions
2. 「New repository secret」をクリック
3. 名前 `ANTHROPIC_API_KEY`、値にAPIキーを貼り付けて保存

ワークフロー側では `${{ secrets.XXX }}` で参照する。

---

## 完全なworkflowファイル

`.github/workflows/daily_pipeline.yml` を以下の内容で作成する。

```yaml
name: Daily Article Pipeline

on:
  schedule:
    - cron: '0 22 * * *'
  workflow_dispatch:

jobs:
  run-pipeline:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run pipeline
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          ZENN_GITHUB_TOKEN: ${{ secrets.ZENN_GITHUB_TOKEN }}
        run: python src/orchestrator.py

      - name: Notify on failure
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `[自動通知] パイプライン失敗 ${new Date().toISOString()}`,
              body: `Run URL: ${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId}`
            })
```

失敗時に自動でIssueを起票することで、メール通知が飛んでくる。Slack通知が欲しい場合は `SLACK_WEBHOOK_URL` をSecretsに追加し、`curl -X POST` で投げるだけでよい。

---

## よくある落とし穴

**UTC時刻のズレを忘れる**
`cron: '0 7 * * *'` と書くとJST16時に動く。本書のパイプラインはJST7時前提のコード（`datetime.now()`比較）が多いため、必ずUTC換算で指定すること。

**`timeout-minutes` を省略するとフリーズで無料枠を食い潰す**
Claude APIの応答待ちでハングすると6時間まるまる消費されることがある。`30` 分を上限に設定しておけば最悪のケースを防げる。

**`pip install` のキャッシュが効かない**
`cache: 'pip'` を指定しているが、`requirements.txt` を更新した直後は必ずキャッシュミスが起きる。初回実行は通常より1〜2分長くかかるため、タイムアウトに余裕を持たせておくこと。

---

以上でcronトリガー・Secrets連携・失敗通知の三点セットが揃う。毎朝7時にActionsタブを開くと実行ログが積み上がっていく様子は、自動化の醍醐味そのものだ。
