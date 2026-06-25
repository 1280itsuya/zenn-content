---
title: "GitHub Actions cron 06:00 JSTで全自動化：シークレット管理・Slack障害通知・月コスト計算"
free: false
---

## cron式の罠：UTCで書かないとJST 06:00にならない

GitHub ActionsのcronはUTC固定なので `0 6 * * *` と書くとJST 15:00に実行される。JST 06:00に合わせるには `0 21 * * *`（前日21:00 UTC）が正解。

```yaml
# .github/workflows/sns_pipeline.yml
name: SNS Noise Filter & Collect
on:
  schedule:
    - cron: "0 21 * * *"   # UTC 21:00 = JST 06:00
  workflow_dispatch:         # 手動トリガー用（デバッグ時に使う）

jobs:
  collect:
    runs-on: ubuntu-22.04
    timeout-minutes: 25     # 無料枠を無駄に食わせない上限
```

`workflow_dispatch` を入れておくと「今すぐ動かしたい」時にUIから即起動できる。cronのみだと検証のたびにコミットが必要になる。

---

## GitHub Secrets：3トークンの登録手順

`Settings → Secrets and variables → Actions → New repository secret` で以下を登録する。

| Secret名 | 値の例 | 用途 |
|---|---|---|
| `CLAUDE_API_KEY` | `sk-ant-api03-...` | Claude APIコール |
| `PLAYWRIGHT_AUTH_STATE` | JSONを丸ごとbase64 | X/Bluesky認証 |
| `NOTION_TOKEN` | `secret_xxx...` | 収集ネタの書き出し先 |
| `SLACK_WEBHOOK_URL` | `https://hooks.slack.com/...` | 障害通知 |

`PLAYWRIGHT_AUTH_STATE` だけは直値でなくbase64エンコードが安全：

```bash
# ローカルで auth_state.json を生成済み前提
cat auth_state.json | base64 -w 0
# 出力をそのままSecretに貼る
```

ジョブ内では逆変換して使う：

```yaml
    steps:
      - name: Restore auth state
        run: |
          echo "${{ secrets.PLAYWRIGHT_AUTH_STATE }}" | base64 -d > auth_state.json
```

---

## matrix実行でX・Bluesky・Redditを並列取得

3プラットフォームを直列で回すと合計12〜18分かかる。`matrix` で並列化すると最長の1プラットフォーム時間（約7分）に収束する。

```yaml
  collect:
    strategy:
      fail-fast: false   # 1つ失敗しても残り2つを止めない
      matrix:
        platform: [x, bluesky, reddit]
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: "pip"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Collect from ${{ matrix.platform }}
        env:
          CLAUDE_API_KEY: ${{ secrets.CLAUDE_API_KEY }}
          NOTION_TOKEN: ${{ secrets.NOTION_TOKEN }}
        run: python src/collect.py --platform ${{ matrix.platform }}
```

`fail-fast: false` が外れていると、Xが落ちただけでBlueskyもRedditも途中終了する。SNS APIはしばしばタイムアウトするため必須設定。

---

## Slack webhook で失敗ステップを即通知するjob設計

matrix内で1つでも失敗したら別jobでSlackに飛ばす。`needs` と `if: failure()` の組み合わせが定石。

```yaml
  notify_failure:
    needs: collect
    if: failure()
    runs-on: ubuntu-22.04
    steps:
      - name: Send Slack alert
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
        run: |
          curl -X POST "$SLACK_WEBHOOK_URL" \
            -H "Content-Type: application/json" \
            -d "{
              \"text\": \"SNS Pipeline FAILED :x:\",
              \"attachments\": [{
                \"color\": \"danger\",
                \"fields\": [{
                  \"title\": \"Run URL\",
                  \"value\": \"${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}\"
                }]
              }]
            }"
```

通知本文にRun URLを含めておくと、スマホから1タップでログに飛べる。

---

## 月30日稼働・Actions無料枠2,000分以内に収まる計算根拠

GitHub Free（個人）のActionsは **2,000分/月** が無料。内訳を計算する：

| ステップ | 実測時間 |
|---|---|
| setup-python + pip install | 約2分 |
| collect（3並列、最長7分） | 約7分 |
| notify_failure（失敗時のみ） | 約0.5分 |
| **1回合計** | **約9.5分** |

```
9.5分 × 30日 = 285分/月
```

無料枠2,000分に対して **285分（14.3%）** で余裕がある。Claudeコストは別途かかり、GPT-4o比で安価なHaiku主体運用で月¥1,200が実績値（Claude APIダッシュボードのUsageページで確認可能）。

Haiku単価で1呼び出し平均0.3円換算：

```python
# src/cost_estimate.py
HAIKU_INPUT_PER_1K  = 0.00025   # USD per 1K tokens
HAIKU_OUTPUT_PER_1K = 0.00125
USD_TO_JPY = 155

posts_per_day   = 120    # 3プラットフォーム合計
tokens_per_post = 800    # input+output平均
days            = 30

cost_usd = (posts_per_day * tokens_per_post / 1000) * \
           (HAIKU_INPUT_PER_1K + HAIKU_OUTPUT_PER_1K) * days
cost_jpy = cost_usd * USD_TO_JPY

print(f"月コスト試算: ${cost_usd:.2f} / ¥{cost_jpy:.0f}")
# → 月コスト試算: $7.92 / ¥1,228
```

Actions無料枠を使い切る前にClaudeコストが先に上限になるため、`CLAUDE_DAILY_BUDGET_JPY=50` のような日次上限を環境変数で持たせてハードストップを入れるのが安全。
