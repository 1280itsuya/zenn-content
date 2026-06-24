---
title: "GitHub Actionsで無人運転化——定時実行・結果通知・ログ管理をゼロコストで構築する"
free: false
---

## GitHub Actionsで無人運転化——定時実行・結果通知・ログ管理をゼロコストで構築する

パイプラインをローカルで動かし続けるのは現実的ではない。PCがスリープすれば止まり、再起動すれば忘れられる。GitHub Actionsのcronジョブを使えば、パブリック・プライベートどちらのリポジトリでも月2,000分まで無料で、クラウド上の定時実行環境を手に入れられる。

---

## ワークフローファイルの基本構造

`.github/workflows/pipeline.yml` を作成する。

```yaml
name: Auto Pipeline

on:
  schedule:
    - cron: '0 22 * * *'   # 毎日UTC22時 = JST翌7時
  workflow_dispatch:         # 手動実行ボタンも残す

jobs:
  run-pipeline:
    runs-on: ubuntu-latest
    timeout-minutes: 30      # ハング防止に必須

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run pipeline
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          SLACK_WEBHOOK:  ${{ secrets.SLACK_WEBHOOK }}
        run: python orchestrator/main.py
```

**落とし穴①：cronのタイムゾーン**  
GitHub ActionsのcronはUTC固定だ。JST7時に実行したければ `0 22 * * *`（前日UTC22時）と書く。`0 7 * * *` と書いてしまうと日本時間の16時に動く。

---

## Slack通知を3行で仕込む

成功・失敗の両方を検知するために、ステップのあとに通知ステップを追加する。

```yaml
      - name: Notify Slack
        if: always()   # 成功・失敗・キャンセルすべてで実行
        run: |
          STATUS="${{ job.status }}"
          COLOR=$([[ "$STATUS" == "success" ]] && echo "good" || echo "danger")
          curl -s -X POST "$SLACK_WEBHOOK" \
            -H 'Content-type: application/json' \
            -d "{\"attachments\":[{\"color\":\"$COLOR\",\"text\":\"Pipeline: $STATUS\"}]}"
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
```

`if: always()` がなければ、パイプラインが失敗したときに通知ステップ自体がスキップされ、無音で終わる。これは最もよくある見落としだ。

Webhookは Slack App の「Incoming Webhooks」から発行し、`https://hooks.slack.com/services/XXX/YYY/ZZZ` 形式のURLをリポジトリの **Settings → Secrets → Actions** に `SLACK_WEBHOOK` として登録する。

---

## ログをアーティファクトとして保存する

```yaml
      - name: Run pipeline
        run: python orchestrator/main.py 2>&1 | tee logs/run.log

      - name: Upload logs
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: pipeline-logs-${{ github.run_number }}
          path: logs/run.log
          retention-days: 14
```

`tee` で標準出力とファイルへ同時に書く。`upload-artifact` でGitHub UI上から14日間ダウンロードできる。エラー調査のたびにSSHで潜り込む必要がなくなる。

**落とし穴②：ログディレクトリが存在しない**  
`logs/` フォルダが空のままリポジトリに入っていないと `tee` が失敗する。`.gitkeep` を置くか、スクリプト側で `os.makedirs("logs", exist_ok=True)` を先頭に書いておく。

---

## 実行時間を節約するキャッシュ設定

`actions/setup-python` に `cache: 'pip'` を指定するだけで、依存ライブラリのインストール時間が初回2分→10秒程度になる。月2,000分の無料枠を消費しにくくなるうえ、起動が速くなるため手動トリガーでのデバッグも快適になる。

---

## 最小動作チェックリスト

- [ ] `cron` の時刻はUTC換算で記述した  
- [ ] `timeout-minutes` を設定してハングを防いだ  
- [ ] Slack通知ステップに `if: always()` を付けた  
- [ ] シークレットはリポジトリのSecrets欄に登録し、コードにハードコードしていない  
- [ ] `logs/` ディレクトリが存在する（`.gitkeep` など）  

これだけ押さえれば、PCの電源を切ったままでもパイプラインが毎朝動き、結果がSlackに届く環境が完成する。
