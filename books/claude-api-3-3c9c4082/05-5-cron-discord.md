---
title: "第5章 cron+構造化ログ+Discord通知で無人運用する監視と失敗アラートの作り込み"
free: false
---

## 結論：無人運用は「止まる前提」で監視層を作る

半年で停止は3回。原因はcron二重起動・コスト暴走・JSONパース例外。先に監視と自動停止を入れたことで、月3万件処理を人手ゼロで維持できた。最初に断言する：**処理ロジックより監視層の作り込みが運用コストを決める**。

## Windowsタスクスケジューラ + cron で日次起動を多重防止する

cron二重起動で同一バッチが走り、1日でAPIコストが¥1,200に膨らんだ。ロックファイルで多重起動を殺す。

```bash
#!/usr/bin/env bash
LOCK=/tmp/pipeline.lock
exec 9>"$LOCK"
flock -n 9 || { echo "already running"; exit 0; }
python -m pipeline.run --date "$(date +%F)"
```

Windowsなら`schtasks /Create /SC DAILY /ST 07:00 /TN pipeline`で7時起動。

## structlog で件数・コスト・失敗率をJSON構造化する

```python
import structlog
log = structlog.get_logger()
log.info("batch_done", processed=29871, failed=42,
         cost_jpy=487.3, fail_rate=round(42/29871, 4))
```

`fail_rate > 0.01`を後段のアラート条件に使う。1行1JSONなので`jq`で即集計できる。

## 日次¥500ガードとチェックポイント再開で損失を止める

コスト超過で1晩¥3,400溶かした事故の恒久対策。処理済みIDを永続化し、上限超過で即停止する。

```python
import json, sys
done = set(json.load(open("done.json")))
if cost_today > 500:        # 日次上限¥500
    log.error("cost_guard_stop", cost=cost_today); sys.exit(1)
for item in items:
    if item.id in done: continue
    process(item); done.add(item.id)
    json.dump(list(done), open("done.json", "w"))
```

途中失敗しても`done.json`から再開でき、再処理コストはゼロになる。

## Discord Webhook で失敗だけを鳴らす

成功通知は無視されるので、失敗率1%超とコストガード作動時のみ飛ばす。

```python
import requests, os
def alert(msg):
    requests.post(os.environ["DISCORD_WEBHOOK"],
                  json={"content": f"⚠️ {msg}"}, timeout=5)
if fail_rate > 0.01:
    alert(f"fail_rate={fail_rate:.2%} cost=¥{cost_today}")
```

`.env`にWebhook URLとAPIキーを置き、コミット対象から除外（`.gitignore`に`.env`）。環境変数経由で読み、平文漏洩を防ぐ。

## 移植チェックリスト：5項目で自分の業務へ載せ替える

```yaml
checklist:
  - flockロックで二重起動を殺したか
  - structlogでcost_jpy/fail_rateをJSON出力しているか
  - done.jsonで処理済みIDを永続化したか
  - 日次¥500のコストガードでsys.exitするか
  - Discord通知は失敗率1%超のみに絞ったか
```

この5項目を満たすと、半年運用で停止3回・うち自動復旧2回まで落ちる。学習用の技術書やVPS選定は[A8.net](https://www.a8.net/)経由の実リンクから比較できる。
