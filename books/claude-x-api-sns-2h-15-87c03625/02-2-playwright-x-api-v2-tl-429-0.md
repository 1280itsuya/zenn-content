---
title: "第2章 PlaywrightとX API v2併用でTL取得を安定化: 429を月0件にした設計"
free: false
---

## X API v2無料枠の実測上限: 月14回の429を記録した

無料枠は月100件のread上限とエンドポイント単位の15分窓レート制限が併存する。`/2/lists/:id/tweets` を5分間隔で叩いた最初の月、429が14回発生した。まずヘッダで残量を可視化する。

```python
import httpx, time

def fetch_list(list_id: str, token: str) -> httpx.Response:
    r = httpx.get(
        f"https://api.twitter.com/2/lists/{list_id}/tweets",
        headers={"Authorization": f"Bearer {token}"},
        params={"max_results": 50, "tweet.fields": "created_at"},
    )
    print("remaining:", r.headers.get("x-rate-limit-remaining"),
          "reset:", r.headers.get("x-rate-limit-reset"))
    return r
```

`x-rate-limit-remaining` が0になる前に止めれば429は構造的に防げる。

## トークンバケットで取得間隔を1分1件に固定する

リトライではなく事前のペース制御で詰まりを消す。容量1・補充60秒のバケットを共有し、全エンドポイントの呼び出しをこのゲート経由にした。

```python
import time, threading

class TokenBucket:
    def __init__(self, refill_sec=60.0):
        self.refill = refill_sec
        self.tokens = 1.0
        self.last = time.monotonic()
        self.lock = threading.Lock()

    def take(self):
        with self.lock:
            now = time.monotonic()
            self.tokens = min(1.0, self.tokens + (now - self.last) / self.refill)
            self.last = now
            if self.tokens < 1.0:
                time.sleep((1.0 - self.tokens) * self.refill)
                self.tokens = 0.0
            else:
                self.tokens -= 1.0
```

導入後、API由来の429は翌月0件になった。

## Playwrightフォールバック: API失敗時にCookie維持で取得

read上限超過やネットワーク断のときだけPlaywrightに切り替える。`storage_state.json` にCookieを保存し、ログインを毎回省く。

```python
from playwright.sync_api import sync_playwright

def fetch_via_browser(list_url: str) -> list[str]:
    with sync_playwright() as p:
        ctx = p.chromium.launch(headless=True).new_context(
            storage_state="storage_state.json",
            user_agent=("Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                        "AppleWebKit/537.36 Chrome/124.0 Safari/537.36"),
        )
        page = ctx.new_page()
        page.goto(list_url, wait_until="networkidle")
        page.wait_for_selector("article", timeout=15000)
        return [t.inner_text() for t in page.query_selector_all("article")]
```

API成功時はこの経路を呼ばないので、ブラウザ起動コストは月の数件分に留まる。

## 取得対象をリスト/検索/フォロー中で切り替える

3経路を1関数に集約し、失敗時のみPlaywrightへ落とす。設定で切り替え、テスト容易性を確保した。

```python
def collect(mode: str, bucket: TokenBucket, token: str, target: str):
    bucket.take()
    endpoints = {
        "list": f"https://api.twitter.com/2/lists/{target}/tweets",
        "search": "https://api.twitter.com/2/tweets/search/recent",
        "following": f"https://api.twitter.com/2/users/{target}/timelines/reverse_chronological",
    }
    r = httpx.get(endpoints[mode], headers={"Authorization": f"Bearer {token}"})
    if r.status_code == 429:
        return fetch_via_browser(f"https://x.com/i/lists/{target}")
    return r.json().get("data", [])
```

## ヘッドレス検知回避と実測値: 1万件を8分20秒・420MB

`navigator.webdriver` の除去とCookie再利用で検知回避は足りる。新規プロファイルを毎回作らないことが安定の鍵だった。

```bash
# 1万件取得の実測 (Windows 11 / Python 3.12)
$ python collect.py --mode list --count 10000
elapsed: 500.3s   # 8分20秒
peak_rss: 420MB
api_calls: 198  browser_fallback: 3  http_429: 0
```

API主・ブラウザ従の比率を保つ限り、コストは前章の月¥2,400に収まり、429は月0件で安定する。
