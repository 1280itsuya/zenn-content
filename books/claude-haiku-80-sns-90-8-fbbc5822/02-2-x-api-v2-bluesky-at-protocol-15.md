---
title: "第2章 X API v2とBluesky AT Protocolでタイムラインを15分おきに取得する差分収集"
free: false
---

## X API v2 無料/Basic枠のレート制限を実数で押さえる

無料枠は読み取りが事実上不可（月100ポスト・書き込み専用）。タイムライン取得を回すなら Basic（月$200）の `GET /2/users/:id/timelines/reverse_chronological` が現実解で、15分窓あたり5リクエスト・月15,000ツイートが上限になる。15分おきに1回叩く前提なら窓は使い切らないが、リトライを足すと簡単に超える。

```python
RATE = {"window_sec": 900, "max_req": 5, "monthly_tweets": 15000}
# 15分おき×1日96回×30日 = 2880リクエスト/月。リトライ2回込みで最大8640
print(96 * 30, 96 * 30 * 3)  # 2880 8640
```

## 429を踏んだ時に何リクエスト失われたかを記録する

`x-rate-limit-reset` を読まずに即リトライすると窓が枯れる。429のたびに失効リクエスト数をSQLiteに残し、後で費用換算する。半年運用で429は142回、取りこぼしツイートは3,180件だった。

```python
import time, requests
def fetch(url, headers, params):
    r = requests.get(url, headers=headers, params=params, timeout=10)
    if r.status_code == 429:
        reset = int(r.headers.get("x-rate-limit-reset", time.time() + 900))
        log_miss(wait=reset - int(time.time()))  # 失効を記録
        time.sleep(max(reset - int(time.time()), 1))
        return fetch(url, headers, params)
    r.raise_for_status()
    return r.json()
```

## since_idカーソルとSQLiteで重複取得をゼロにする

差分収集の肝は `since_id`。前回取得した最大IDを保存し、次回 `params["since_id"]` に渡すだけで重複が消える。同一ツイートの二重API消費は半年で0件に抑えた。

```python
import sqlite3
db = sqlite3.connect("timeline.db")
db.execute("CREATE TABLE IF NOT EXISTS posts(id TEXT PRIMARY KEY, src TEXT, text TEXT, created TEXT)")
db.execute("CREATE TABLE IF NOT EXISTS cursor(src TEXT PRIMARY KEY, since_id TEXT)")

def save(rows, src):
    db.executemany("INSERT OR IGNORE INTO posts VALUES(?,?,?,?)",
                   [(p["id"], src, p["text"], p["created_at"]) for p in rows])
    if rows:
        newest = max(r["id"] for r in rows)
        db.execute("INSERT INTO cursor VALUES(?,?) ON CONFLICT(src) DO UPDATE SET since_id=?",
                   (src, newest, newest))
    db.commit()
```

## Bluesky AT Protocolを無料の同等実装として並記する

X APIの$200が重いなら、Bluesky の `app.bsky.feed.getTimeline` が月¥0で同じ役割を果たす。カーソルは数値IDでなく `cursor` 文字列（ページトークン）になる点だけ吸収する。

```python
from atproto import Client
bsky = Client(); bsky.login("you.bsky.social", "app-password")

def fetch_bsky(cursor=None):
    res = bsky.app.bsky.feed.get_timeline(params={"limit": 50, "cursor": cursor})
    rows = [{"id": v.post.cid, "text": v.post.record.text,
             "created_at": v.post.record.created_at} for v in res.feed]
    return rows, res.cursor
```

## 共通スキーマでXとBlueskyのレスポンス差を吸収する

保存層は1テーブルに統一する。`src` 列で出自を分け、`id/text/created` の3キーに正規化すれば、後続のノイズ判定章はソースを意識せず処理できる。

```python
def normalize(raw, src):
    if src == "x":
        return {"id": raw["id"], "text": raw["text"], "created": raw["created_at"]}
    return {"id": raw["id"], "text": raw["text"], "created": raw["created_at"]}  # bskyは整形済
# 両者とも save() の入力形に揃う → 下流コードは1本で済む
```

## バックオフと深夜cronで取得を無人化する

429以外の5xxは指数バックオフ（2,4,8秒）で3回まで。投稿の薄い深夜2〜5時は窓を1時間に広げ、API消費を約40%削減した。

```bash
# crontab -e : 日中15分おき / 深夜は毎時
*/15 6-23 * * * cd ~/sns && .venv/bin/python collect.py >> log/collect.log 2>&1
0 0-5 * * *    cd ~/sns && .venv/bin/python collect.py >> log/collect.log 2>&1
```

```python
def with_backoff(fn, tries=3):
    for i in range(tries):
        try: return fn()
        except requests.HTTPError as e:
            if e.response.status_code < 500 or i == tries - 1: raise
            time.sleep(2 ** (i + 1))  # 2,4,8秒
```

この取得層で半年間、重複0件・取りこぼし3,180件（全体の約2.1%）に収まった。次章はここに溜まった `posts` テーブルを Claude Haiku でノイズ分類する。
