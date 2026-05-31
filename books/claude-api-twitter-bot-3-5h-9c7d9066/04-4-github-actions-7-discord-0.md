---
title: "第4章: GitHub Actionsで毎朝7時に無人実行しDiscordへ要約通知する月¥0運用パイプライン"
free: false
---

第4章の本文です。

---

先に結論を伝える。ローカルの`python mute_bot.py`手動実行をやめ、GitHub Actionsのcronで毎朝7時(JST)に無人実行する。1回の実行は約90秒、月30回で45分。Actions無料枠の月2,000分に対して2.3%しか食わない。APIもActionsもストレージも¥0で回る。

## GitHub Secretsへ4キーを登録しYAMLから参照する

`X_BEARER_TOKEN`・`ANTHROPIC_API_KEY`・`DISCORD_WEBHOOK`・`MUTED_STATE_GIST_ID`の4つをリポジトリの`Settings > Secrets and variables > Actions`へ登録する。CLIなら一括投入が速い。

```bash
gh secret set X_BEARER_TOKEN --body "$X_BEARER_TOKEN"
gh secret set ANTHROPIC_API_KEY --body "$ANTHROPIC_API_KEY"
gh secret set DISCORD_WEBHOOK --body "$DISCORD_WEBHOOK"
gh secret set MUTED_STATE_GIST_ID --body "your_gist_id"
```

平文をworkflowにベタ書きすると公開リポジトリでBotがアカウントごと乗っ取られる。Secrets経由が必須。

## cron `0 22 * * *` でJST 7時に起動するworkflow.yml全文

GitHub ActionsのcronはUTC基準なので、JST 7時はUTC 22時。`0 22 * * *`と書く。

```yaml
name: mute-bot
on:
  schedule:
    - cron: "0 22 * * *"
  workflow_dispatch:
jobs:
  run:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt
      - name: run mute bot
        env:
          X_BEARER_TOKEN: ${{ secrets.X_BEARER_TOKEN }}
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          DISCORD_WEBHOOK: ${{ secrets.DISCORD_WEBHOOK }}
        run: python mute_bot.py --limit 50
```

`workflow_dispatch`を入れておくと、手動の即時テストがActions画面のボタン1つで打てる。

## Discord Webhookへ「ミュート済み/要確認」を2系統で要約通知する

Claude Haikuのスコアが0.8以上は自動ミュート、0.5〜0.8は誤爆防止のため「要確認」に振り分けてEmbedで色分け通知する。

```python
import os, requests

def notify(muted: list[str], review: list[str]):
    embeds = [
        {"title": f"✅ 自動ミュート {len(muted)}件", "color": 3066993,
         "description": "\n".join(muted[:10]) or "なし"},
        {"title": f"⚠️ 要確認 {len(review)}件", "color": 15844367,
         "description": "\n".join(review[:10]) or "なし"},
    ]
    requests.post(os.environ["DISCORD_WEBHOOK"],
                  json={"embeds": embeds}, timeout=10)
```

毎朝スマホのDiscordに2件のEmbedが届くだけ。判定ログを開く手間が消え、確認は30秒で終わる。

## 月¥0の内訳と無料枠超過時の`--limit`間引き

実測コストを合算する。X API freeは月1,500件取得・¥0。Claude Haikuは50件×30日=1,500件で入力約75万トークン、$0.80/Mトークン換算で月約$0.06だが、Anthropicの無料クレジット内なら実質¥0。Actionsは45分/2,000分、状態保存はGist(無料)。すべて¥0で成立する。

枠が逼迫したら`--limit`を絞る。X APIの月1,500件に近づいたら1回25件へ半減すれば月750件で収まる。

```python
import argparse
p = argparse.ArgumentParser()
p.add_argument("--limit", type=int, default=50)
args = p.parse_args()
monthly_budget = 1500
per_run = min(args.limit, monthly_budget // 30)  # 1日上限を自動算出
```

`per_run`を自動算出にしておけば、月末に枠を使い切って通知が止まる事故を防げる。週3.5h削減の運用を、課金リスクゼロで無人化できる。

---

topics: [claude, ai, python, twitter, automation]
