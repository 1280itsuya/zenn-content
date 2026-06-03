---
title: "第2章 X API v2 Free枠の月1万投稿上限を超えないtweepy設計とレート制限429の回避コード"
free: false
---

結論から言うと、X API v2 Free枠は「投稿read月1万件」が天井であり、home timelineを素直に叩くと数日で枯渇する。この章では消費投稿数を見える化し、1回の収集を最小化するtweepy設計と429回避コードを実装する。

## X API v2 Free枠の月1万read上限を消費投稿数からカウントする

2023年の改悪でFree枠はread相当が月1万件まで縮小した。どのエンドポイントが何件消費するかを先に概算する。

```python
# 1リクエストあたりのmax_results = そのまま消費read数
ENDPOINT_COST = {
    "search_recent_tweets": 100,  # 1リクエスト最大100件
    "get_home_timeline":    100,
}
MONTHLY_READ_LIMIT = 10_000

def remaining_requests(used_reads: int, endpoint: str) -> int:
    return (MONTHLY_READ_LIMIT - used_reads) // ENDPOINT_COST[endpoint]

print(remaining_requests(0, "get_home_timeline"))  # -> 100 (月100リクエストで枯渇)
```

月100リクエストしか撃てない。90分ごとのポーリングでは即死するため、消費を絞る設計が前提になる。

## tweepy.Client(wait_on_rate_limit=True)とsince_id差分取得で消費を1/5にする

毎回全件を取り直すのが最大の無駄である。`since_id`で前回以降の差分だけを取得する。

```python
import tweepy

client = tweepy.Client(
    bearer_token=BEARER,
    wait_on_rate_limit=True,  # 429到達時にtweepyがreset時刻まで自動待機
)

def fetch_diff(last_id: int | None) -> tuple[list, int | None]:
    resp = client.get_home_timeline(
        since_id=last_id,          # 前回の最大IDより新しいものだけ
        max_results=100,
        tweet_fields=["author_id", "created_at"],
    )
    tweets = resp.data or []
    newest = tweets[0].id if tweets else last_id
    return tweets, newest
```

`since_id`未指定だと毎回100件フル消費するが、差分なら静かな時間帯は数件で済み、実測で月消費が約1/5に落ちる。

## 429/503をexponential backoffデコレータでリトライする

`wait_on_rate_limit`はレート上限の待機だけで、503(サーバ過負荷)はカバーしない。両方をデコレータで包む。

```python
import time, functools
from tweepy.errors import TooManyRequests, TwitterServerError

def retry_backoff(max_tries=5, base=2):
    def deco(fn):
        @functools.wraps(fn)
        def wrap(*a, **kw):
            for n in range(max_tries):
                try:
                    return fn(*a, **kw)
                except (TooManyRequests, TwitterServerError) as e:
                    wait = base ** n  # 1,2,4,8,16秒
                    print(f"retry {n+1}: {type(e).__name__} -> {wait}s")
                    time.sleep(wait)
            raise RuntimeError("max retries exceeded")
        return wrap
    return deco

@retry_backoff()
def safe_fetch(last_id):
    return fetch_diff(last_id)
```

## Paginatorのlimitで月1万readを超えない打ち切り設計

`Paginator`を素で回すと取れるだけ取って上限を食い潰す。`limit`でハードストップを置く。

```python
def collect(last_id, budget_reads=300):
    pages = budget_reads // 100  # この収集で使うreadを予算化
    collected = []
    for page in tweepy.Paginator(
        client.get_home_timeline,
        since_id=last_id,
        max_results=100,
        limit=pages,               # 最大pages回でハードストップ
    ):
        collected.extend(page.data or [])
    return collected
```

`budget_reads=300`なら1回の収集を確実に3リクエスト以内に抑え、月1万readの枠内で日次運用が回る。差分取得とこの予算化により、90分かけていた全件ポーリングが18分・低消費の差分収集へ置き換わる。
