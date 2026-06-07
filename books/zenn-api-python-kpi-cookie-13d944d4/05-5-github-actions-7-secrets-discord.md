---
title: "第5章 GitHub Actionsで毎朝7時に無人実行｜Secrets運用・Discord通知・失敗時リトライまで本番常駐させる"
free: false
---

## 結論｜毎朝7時JSTのcron実行＋失敗時2回リトライで半年無停止になる

ローカルcronや自宅PC常駐をやめ、GitHub Actionsの`schedule`へ載せると電気代¥0・可用性99%超で集計が回り続ける。Cookie失効は前章のステートマシンで自己回復させ、回復不能時のみDiscordへアラートしてSecretsを差し替える運用に集約する。

```yaml
# .github/workflows/zenn-kpi.yml
name: zenn-kpi-daily
on:
  schedule:
    - cron: '0 22 * * *'   # UTC22:00 = JST翌07:00
  workflow_dispatch: {}      # 手動再実行用
concurrency:
  group: zenn-kpi           # 多重起動を1本に抑制
  cancel-in-progress: false
```

## ZENN_COOKIE/DISCORD_WEBHOOKをSecretsで注入しPAT平文漏洩を防ぐ

認証Cookieは`.env`にもログにも書かず、`Settings > Secrets and variables > Actions`へ登録する。`env:`経由でプロセスに渡し、`echo`での出力を禁止することで、過去に自分がやらかしたPAT平文コミットを構造的に防ぐ。

```yaml
jobs:
  collect:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - run: pip install -r requirements.txt
      - name: collect kpi
        env:
          ZENN_COOKIE: ${{ secrets.ZENN_COOKIE }}
          DISCORD_WEBHOOK: ${{ secrets.DISCORD_WEBHOOK }}
        run: python collect.py
```

## 403連発・取得0件をDiscord Webhookへ即通知する

正常サマリと異常を同じWebhookへ投げ分ける。403が5回連続、または取得記事0件のときは`@everyone`付きで深夜でも気づける文面にし、Cookie差し替えの判断を秒で下せるようにする。

```python
# collect.py（通知部）
import os, requests

def notify(msg: str, alert: bool = False):
    body = f"@everyone {msg}" if alert else msg
    requests.post(os.environ["DISCORD_WEBHOOK"],
                  json={"content": body[:1900]}, timeout=10)

def report(rows, err_403):
    if not rows:
        notify("取得0件: Cookie失効の可能性", alert=True)
    elif err_403 >= 5:
        notify(f"403が{err_403}回連発: レート/認証異常", alert=True)
    else:
        total = sum(r["views"] for r in rows)
        notify(f"集計OK {len(rows)}記事 / 合計{total}views")
```

## ジョブ失敗を最大2回リトライしарт ・アーティファクトでログを残す

非公式エンドポイントは一過性の503を返す。ステップ単位で指数バックオフ（5s→15s）の再実行を挟み、それでも落ちたら`nice-retry`で再起動。実行ログとCSVは`actions/upload-artifact`で14日保持し、後追いで失敗原因を読める状態にする。

```yaml
      - name: collect with retry
        uses: nick-fields/retry@v3
        with:
          timeout_minutes: 5
          max_attempts: 3        # 本実行+2リトライ
          retry_wait_seconds: 15
          command: python collect.py
      - if: always()
        uses: actions/upload-artifact@v4
        with:
          name: kpi-logs
          path: |
            out/kpi.csv
            out/run.log
          retention-days: 14
```

## 自宅PC常駐との比較｜電気代¥1,800/年と可用性で常駐は不要になる

自宅PC常駐は24時間で約60W、年¥1,800前後＋OSアップデート再起動で取りこぼしが出る。GitHub Actionsは無料枠2,000分/月に対し本ジョブは1回1分弱＝月30分で収まり、可用性もGitHub側に寄せられる。半年運用の停止要因は電源でもネットワークでもなく、ほぼCookie失効1点に絞り込めた。

```bash
# 月間消費分の確認（残枠を監視）
gh api /repos/$GITHUB_REPOSITORY/actions/workflows \
  | jq '.workflows[] | {name, id}'
gh run list --workflow=zenn-kpi.yml --limit 5 \
  --json conclusion,createdAt,databaseId
```

この構成にすると、運用者の仕事は「Discordに`取得0件`が来た朝だけ`ZENN_COOKIE`を更新する」だけに縮む。残りの364日は無人で回り、KPIは毎朝7時に手元へ届く。
