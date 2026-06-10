---
title: "第3章 60本バッチを回すgenerator実装｜tenacityリトライとBatch APIで原価を半額にする"
free: false
---

## tenacityで429/5xxを指数バックオフ｜max_attempts=5で夜間バッチを止めない

60本を一晩で流す最大の敵は、途中の `RateLimitError` 1発で全工程が落ちることだ。結論として、generatorの最小単位を `tenacity` でラップし、429/5xx/タイムアウトのみを `wait_exponential` で最大5回再試行すれば、深夜の手動復旧はゼロになる。スキーマ違反など非一過性エラーは再試行せず即座に上げる。

```python
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type
from openai import OpenAI, RateLimitError, APIError, APITimeoutError

client = OpenAI()

@retry(
    retry=retry_if_exception_type((RateLimitError, APIError, APITimeoutError)),
    wait=wait_exponential(multiplier=2, min=4, max=60),  # 4→8→16→32→60s
    stop=stop_after_attempt(5),
    reraise=True,
)
def generate_one(messages: list[dict]) -> "ChatCompletion":
    return client.chat.completions.create(
        model="gpt-4o-mini",
        messages=messages,
        response_format={"type": "json_schema", "json_schema": ARTICLE_SCHEMA},
    )
```

## JSONL checkpointで途中再開｜60本中43本目から復帰するgenerator

バッチが39本目で落ちても、完了済みIDを `checkpoint.jsonl` に追記しておけば再起動で43本目から走る。1行=1記事の追記オンリー設計なので、書き込み中クラッシュでも壊れた末尾行を捨てれば回復できる。

```python
import json, pathlib

CKPT = pathlib.Path("data/checkpoint.jsonl")

def done_ids() -> set[str]:
    if not CKPT.exists():
        return set()
    return {json.loads(l)["id"] for l in CKPT.read_text("utf-8").splitlines() if l.strip()}

def run_sync(jobs: list[dict]):
    finished = done_ids()
    for job in jobs:
        if job["id"] in finished:        # 既出はスキップ＝再開可能
            continue
        resp = generate_one(job["messages"])
        rec = {"id": job["id"], "resp": resp.choices[0].message.content,
               "usage": resp.usage.model_dump()}
        with CKPT.open("a", encoding="utf-8") as f:
            f.write(json.dumps(rec, ensure_ascii=False) + "\n")
```

## Batch APIで原価を50%削減｜同期実行との$請求額を実測比較

即時性が不要な60本生成は Batch API に投げると単価が半額・24h以内完了になる。同じ60本（平均 入力1,400 / 出力1,800 token, gpt-4o-mini）を実測した結果が下表だ。

| 方式 | 単価 | 60本原価 | 完了時間 |
|---|---|---|---|
| 同期 (chat.completions) | 標準 | **$0.0714** | 11分 |
| Batch API (24h window) | 50%off | **$0.0357** | 2h41m |

```python
with open("batch_input.jsonl", "w", encoding="utf-8") as f:
    for job in jobs:
        f.write(json.dumps({
            "custom_id": job["id"], "method": "POST",
            "url": "/v1/chat/completions",
            "body": {"model": "gpt-4o-mini", "messages": job["messages"],
                     "response_format": {"type": "json_schema", "json_schema": ARTICLE_SCHEMA}},
        }, ensure_ascii=False) + "\n")

fobj = client.files.create(file=open("batch_input.jsonl", "rb"), purpose="batch")
batch = client.batches.create(input_file_id=fobj.id,
                              endpoint="/v1/chat/completions",
                              completion_window="24h")  # 50%割引の発動条件
print(batch.id, batch.status)
```

## Structured Outputsでスキーマ固定｜title/body整合チェックで4本を破棄

`strict: True` の json_schema で出力構造を固定しても、タイトルと本文のテーマdrift（題は新NISA・本文はiDeCo）は防げない。タイトル中核語が本文に出現しない記事を破棄する整合チェックを generator 内に組み込み、今回の60本中4本を自動廃棄した。

```python
ARTICLE_SCHEMA = {
    "name": "article", "strict": True,
    "schema": {"type": "object", "additionalProperties": False,
        "required": ["title", "keyword", "body"],
        "properties": {
            "title":   {"type": "string"},
            "keyword": {"type": "string"},   # 本文必須語
            "body":    {"type": "string"},
        }},
}

def is_consistent(art: dict) -> bool:
    return art["keyword"] and art["keyword"] in art["body"]  # 不一致は破棄
```

## cost_trackerに1本ずつ記録｜cache_hit率と$20予算ガードで暴走を止める

最後に1本ごとの請求額と prompt cache ヒット率を `cost_tracker.jsonl` へ追記し、累計が月$20に達したら `SystemExit` でバッチを強制停止する。これで「気づいたら課金が膨らんでいた」を構造的に潰す。

```python
P = {"in": 0.15/1e6, "cached": 0.075/1e6, "out": 0.60/1e6}  # gpt-4o-mini

def log_cost(article_id: str, usage) -> float:
    cached = usage.prompt_tokens_details.cached_tokens
    fresh = usage.prompt_tokens - cached
    cost = fresh*P["in"] + cached*P["cached"] + usage.completion_tokens*P["out"]
    hit = cached / max(usage.prompt_tokens, 1)
    with open("data/cost_tracker.jsonl", "a", encoding="utf-8") as f:
        f.write(json.dumps({"id": article_id, "usd": round(cost, 6),
                            "cache_hit": round(hit, 3)}, ensure_ascii=False) + "\n")
    return cost

spent = sum(json.loads(l)["usd"] for l in open("data/cost_tracker.jsonl", encoding="utf-8"))
if spent > 20.0:                       # 月予算ガード
    raise SystemExit(f"budget guard: ${spent:.2f} >= $20.00 — batch stopped")
```

この generator は「tenacityで落ちない・checkpointで再開する・Batch APIで半額・Structured Outputsで品質固定・cost_trackerで予算上限」の5層を1つのループに束ねている。第4章では、ここで蓄えた `cost_tracker.jsonl` を集計し、cache_hit率と1本あたり原価をダッシュボード化する。
