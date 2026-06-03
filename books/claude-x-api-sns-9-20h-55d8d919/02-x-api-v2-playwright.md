---
title: "X API v2無料枠の実測上限とPlaywrightフォールバック"
free: false
---

## Freeプランの実測上限：月100件・15分0リクエストの壁

X API v2のFreeプランは2026年6月時点で読み取りが事実上封鎖されている。`tweet.fields`を指定しても`GET /2/tweets/search/recent`は403を返す。実測値を表で開示する。

| 項目 | Free | Basic($200/月) |
|---|---|---|
| 月間Pull上限 | 100 tweets | 15,000 tweets |
| search/recent | 403 | 利用可 |
| users/:id/tweets | 1 req/15min | 5 req/15min |
| 取得フィールド | id,text | +public_metrics,created_at |

```bash
curl -s -H "Authorization: Bearer $X_BEARER" \
  "https://api.twitter.com/2/users/me/tweets?max_results=5&tweet.fields=public_metrics" \
  | python -c "import sys,json;d=json.load(sys.stdin);print(d.get('status') or len(d.get('data',[])))"
# Free → 200だが月100件で打ち止め / Basic → public_metrics付きで取得
```

## 429を避けるウィンドウ設計：900秒間隔・5件バッチ

15分=900秒で1リクエストが上限のため、取得間隔を901秒、1バッチ5件に固定すると月100件枠を約20日に分散できる。`Retry-After`ヘッダを必ず読む。

```python
import time, requests

def pull(uid, token, batch=5, window=901):
    while True:
        r = requests.get(
            f"https://api.twitter.com/2/users/{uid}/tweets",
            headers={"Authorization": f"Bearer {token}"},
            params={"max_results": batch, "tweet.fields": "public_metrics,created_at"},
        )
        if r.status_code == 429:
            wait = int(r.headers.get("Retry-After", window))
            time.sleep(wait); continue
        return r.json().get("data", [])
```

## Playwrightフォールバック：自分のTL取得に限定する運用ガード

API枠を使い切ったら、ログイン済みセッションをPlaywrightで開いて自分のホームTLのみをスクレイプする。検知回避はしない。アクセス先を`/home`に固定し、他人のプロフィール巡回を物理的に禁じるガードを入れる。

```python
from playwright.sync_api import sync_playwright

ALLOW = "https://x.com/home"  # 自TLのみ。他URLは即中止

def scrape_own_tl(state_path="x_state.json"):
    with sync_playwright() as p:
        ctx = p.chromium.launch(headless=True).new_context(storage_state=state_path)
        page = ctx.new_page()
        page.goto(ALLOW, wait_until="networkidle")
        if not page.url.startswith(ALLOW):
            raise RuntimeError("guard: 自TL以外へのアクセスを中止")
        return page.locator('article[data-testid="tweet"]').all_inner_texts()
```

## 正規化スキーマ：APIとHTMLを同一の4キーに揃える

API由来とPlaywright由来でフィールド差があるため、`tweet_id/author/body/metrics`の4キーに正規化し`source`で出所を残す。1万件をClaude Haikuで分類しても¥48に収まる粒度はこの構造で確定する。

```python
from dataclasses import dataclass, asdict

@dataclass
class Tweet:
    tweet_id: str
    author: str
    body: str
    metrics: dict          # {"like":0,"reply":0,"retweet":0}
    source: str            # "api_v2" | "playwright"

def from_api(d):
    m = d.get("public_metrics", {})
    return asdict(Tweet(d["id"], d.get("author_id",""), d["text"],
        {"like": m.get("like_count",0), "reply": m.get("reply_count",0),
         "retweet": m.get("retweet_count",0)}, "api_v2"))
```

この4キーをそのまま次章の分類器へ渡せば、API枠が枯れてPlaywrightへ切り替わっても下流コードは無改修で動く。
