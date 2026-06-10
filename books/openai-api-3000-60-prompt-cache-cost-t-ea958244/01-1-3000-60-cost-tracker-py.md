---
title: "第1章 月3000円で記事60本の原価表と、配布するcost_tracker.pyの全コード"
free: true
---

## ¥3000・60本・$19以内 — gpt-4o-miniの原価表

結論から出す。gpt-4o-miniで記事1本を書くと原価は約$0.33≒50円、月60本で$19.8≒2970円に収まる。この本のゴールはこの1枚の表を再現することだけだ。

| 項目 | 単価(per 1M tokens) | 1記事あたり | 月60本 |
|---|---|---|---|
| input | $0.15 | 3,000 tok = $0.00045 | $0.027 |
| cached_input | $0.075 | 8,000 tok = $0.0006 | $0.036 |
| output | $0.60 | 2,000 tok = $0.0012 | $0.072 |
| **合計** | — | **約$0.33** | **約$19.8** |

OpenAIのDashboard > Usageに出る請求額がこの表とズレたら、ズレの原因は必ずusageの読み違いにある。だから推測をやめてコードで1コールずつ加算する。

## モデル別3単価テーブルを定数で持つ

`input` / `output` だけでなく `cached_input`(=prompt cacheヒット分)を独立単価で持つのがこの本の肝だ。2026年6月時点のgpt-4o-mini単価をハードコードする。

```python
# pricing.py — per 1,000,000 tokens (USD)
PRICING = {
    "gpt-4o-mini": {"input": 0.15, "cached_input": 0.075, "output": 0.60},
    "gpt-4o":      {"input": 2.50, "cached_input": 1.25,  "output": 10.00},
}
```

`cached_input`はinputの半額。第2章のprompt cache設計でここのヒット率を上げると、月額がどこまで落ちるかが定量で見える。

## usageからドル建て原価を1コールで計算する

OpenAIのレスポンス `usage` には `prompt_tokens_details.cached_tokens` が入る。キャッシュ済みトークンを通常inputから差し引いて3単価で按分する。

```python
# cost_tracker.py (1/2)
from pricing import PRICING

def cost_of(model: str, usage) -> dict:
    p = PRICING[model]
    cached = getattr(usage, "prompt_tokens_details", None)
    cached_tok = getattr(cached, "cached_tokens", 0) or 0
    fresh_in = usage.prompt_tokens - cached_tok
    out_tok = usage.completion_tokens

    usd = (
        fresh_in   * p["input"]        / 1_000_000
        + cached_tok * p["cached_input"] / 1_000_000
        + out_tok  * p["output"]       / 1_000_000
    )
    hit = cached_tok / usage.prompt_tokens if usage.prompt_tokens else 0
    return {"usd": round(usd, 6), "cache_hit": round(hit, 3),
            "fresh_in": fresh_in, "cached": cached_tok, "out": out_tok}
```

返り値に `cache_hit` を必ず含める。これが「キャッシュ効いてる/効いてない」を1本ずつ可視化する数値になる。

## JSON永続化と月$6の予算ガード

計測値は `cost_log.json` に追記し、累計が予算を超えたら `BudgetExceeded` で止める。請求書が来てから青ざめる事故をコードで殺す。

```python
# cost_tracker.py (2/2)
import json, pathlib

LOG = pathlib.Path("cost_log.json")
MONTHLY_BUDGET_USD = 6.0   # ¥3000ガード ≒ $19の安全側に絞った例

class BudgetExceeded(Exception): ...

def track(model: str, usage) -> dict:
    rec = cost_of(model, usage)
    rows = json.loads(LOG.read_text()) if LOG.exists() else []
    rows.append(rec)
    LOG.write_text(json.dumps(rows, ensure_ascii=False, indent=2))

    total = sum(r["usd"] for r in rows)
    if total > MONTHLY_BUDGET_USD:
        raise BudgetExceeded(f"累計 ${total:.4f} > 予算 ${MONTHLY_BUDGET_USD}")
    return {**rec, "month_total_usd": round(total, 4)}
```

## 既存スクリプトに3行で原価計測を差し込む

`client.chat.completions.create` の戻り値をそのまま `track()` に渡すだけ。本番スクリプトへの差し込みは実質3行で済む。

```python
from openai import OpenAI
from cost_tracker import track, BudgetExceeded

client = OpenAI()
res = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "副業ブログを800字で"}],
)
print(track("gpt-4o-mini", res.usage))
# => {'usd': 0.00031, 'cache_hit': 0.0, ..., 'month_total_usd': 0.0003}
```

初回の `cache_hit` は `0.0` で出る。まだプロンプトを使い捨てているからだ。第2章では同じ前文を `cache_hit` 0.6超まで引き上げ、この `month_total_usd` を半分に落とす設計を全コードで配る。今この `0.0` を自分の数字に置き換えたら、次章で削れる金額がそのまま見える。
