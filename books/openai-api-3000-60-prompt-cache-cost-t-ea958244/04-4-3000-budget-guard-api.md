---
title: "第4章 月3000円を絶対に超えないbudget_guard｜上限到達でAPIを止める実装と通知"
free: false
---

第4章 本文:

## budget_guardで月3000円(=$20)を二段しきい値で止める設計

budget_guardは「事後に気づく」cost_trackerを「事前に止める」安全弁へ昇格させる仕組みだ。月初に累計をリセットし、日次$1.0・月次$20.0の二段しきい値を持たせる。80%で警告、100%で`BudgetExceeded`例外を投げてAPIコール自体を中断する。

第3章のcost_trackerが書き出す`cost_log.jsonl`(1コール1行・USD建て)をそのまま入力に使う。

```python
# budget_guard.py
import json, datetime
from pathlib import Path

LOG = Path("cost_log.jsonl")
DAILY_USD, MONTHLY_USD = 1.0, 20.0  # 月3000円≒$20

class BudgetExceeded(Exception): pass

def _spent():
    today = datetime.date.today()
    d = m = 0.0
    for line in LOG.read_text(encoding="utf-8").splitlines():
        r = json.loads(line)
        ts = datetime.date.fromisoformat(r["date"])
        if ts.year == today.year and ts.month == today.month:
            m += r["usd"]
            if ts == today: d += r["usd"]
    return d, m
```

## 80%警告・100%停止をデコレータでOpenAI呼び出し前に挟む

しきい値判定はAPIを叩く直前に置く。100%超なら例外で停止、80%超なら通知して継続する。デコレータ化すれば既存の生成関数に1行追加するだけで全コールへ波及する。

```python
from functools import wraps

def budget_guard(fn):
    @wraps(fn)
    def wrapper(*a, **kw):
        d, m = _spent()
        if d >= DAILY_USD or m >= MONTHLY_USD:
            raise BudgetExceeded(f"day=${d:.2f} month=${m:.2f} 到達—API停止")
        if m >= MONTHLY_USD * 0.8:
            notify(f"⚠ 月予算80%: ${m:.2f}/{MONTHLY_USD}")
        return fn(*a, **kw)
    return wrapper

@budget_guard
def generate_article(prompt: str) -> str:
    ...  # 第2章のclient.chat.completions.create呼び出し
```

## Discord Webhookへ80%警告を5分デバウンス付きで送る

80%警告が毎コール飛ぶと通知が60本分洪水になる。`notify_state.json`に最終送信時刻を記録し、300秒(5分)以内の再送を抑止する。Slack Incoming Webhookも`content`キーを`{"text":...}`へ変えるだけで同形で動く。

```python
import time, requests
WEBHOOK = "https://discord.com/api/webhooks/xxxx"
STATE = Path("notify_state.json")

def notify(msg: str, debounce=300):
    last = json.loads(STATE.read_text()) if STATE.exists() else {"t": 0}
    if time.time() - last["t"] < debounce:
        return
    requests.post(WEBHOOK, json={"content": msg}, timeout=10)
    STATE.write_text(json.dumps({"t": time.time()}))
```

## OpenAI Costs APIの実請求額と自前計測値のズレを補正する

自前計測は無料枠・端数・prompt cache割引を取りこぼし、実請求と数%ズレる。OpenAIの`/v1/organization/costs`から日次の確定USDを引き、自前合計との比率`factor`を求めてしきい値判定へ掛ける。これで「自前では$18なのに実請求$20.5で超過」という事故を防ぐ。

```python
def reconcile(month_start_unix: int) -> float:
    r = requests.get(
        "https://api.openai.com/v1/organization/costs",
        headers={"Authorization": f"Bearer {ADMIN_KEY}"},
        params={"start_time": month_start_unix, "bucket_width": "1d"},
        timeout=20,
    ).json()
    actual = sum(b["results"][0]["amount"]["value"]
                 for b in r["data"] if b["results"])
    _, mine = _spent()
    factor = actual / mine if mine else 1.0
    return round(factor, 3)  # 例: 1.07 → 自前×1.07で判定
```

## cronで深夜2時に暴走を止め、月初にカウンタをリセットする

無人運用の最後の砦はcron側の停止フローだ。生成バッチ起動前に`budget_guard`を素通りチェックし、超過なら`exit 3`してOpenAIを一切呼ばない。月初(1日)だけ`cost_log.jsonl`を退避リネームして累計を$0へ戻す。

```bash
# crontab: 毎日2:00生成 / 月初0:05リセット
0 2 * * * cd /opt/auto && python -c "import budget_guard as b; \
  d,m=b._spent(); exit(3) if m>=b.MONTHLY_USD else None" \
  && python run_batch.py >> batch.log 2>&1
5 0 1 * * mv /opt/auto/cost_log.jsonl \
  /opt/auto/archive/cost_$(date +\%Y\%m).jsonl
```

`exit 3`を`&&`の前段に置くのが要点で、超過月は`run_batch.py`へ到達しない。深夜に60本を一気生成して$20を突き破る事故を、コード・通知・cronの三重で封じられる。
