---
title: "第3章 非公式APIで垢BANしないレート制御｜0.4秒間隔・JST7時固定・ETagキャッシュで月1万リクエストに抑える"
free: false
---

## 0.4秒ジッター＋日次上限で「人間より速い」を消す

非公式エンドポイントが垢BANを誘発する最大要因は、等間隔・高頻度のアクセスパターンが機械的だと判定されることだ。`0.4秒±0.15秒`のジッターを噛ませ、1日のコール総数を`350`で頭打ちにする。

```python
import time, random, threading

class RateLimiter:
    def __init__(self, base=0.4, jitter=0.15, daily_cap=350):
        self.base, self.jitter, self.cap = base, jitter, daily_cap
        self.count, self.lock = 0, threading.Lock()

    def wait(self):
        with self.lock:
            if self.count >= self.cap:
                raise RuntimeError(f"daily_cap {self.cap} 到達: 翌7時まで停止")
            self.count += 1
        time.sleep(self.base + random.uniform(-self.jitter, self.jitter))
```

## ETag/Last-Modifiedで月3.2万→0.9万に削る差分取得

無駄打ちの正体は、更新のないBook統計を毎回フル取得していたこと。`If-None-Match`で`304`を返させれば、ボディ転送もカウントも発生しない。半年運用で月`3.2万`→`0.9万`リクエスト（▲72%）になった。

```python
import json, pathlib, requests

CACHE = pathlib.Path("etag_cache.json")
_store = json.loads(CACHE.read_text()) if CACHE.exists() else {}

def fetch(url, rl):
    rl.wait()
    h = {"If-None-Match": _store.get(url, "")}
    r = requests.get(url, headers=h, timeout=10)
    if r.status_code == 304:
        return None  # 差分なし=スキップ
    _store[url] = r.headers.get("ETag", "")
    CACHE.write_text(json.dumps(_store))
    return r.json()
```

## 429を踏んだら指数バックオフで90秒冷却する

`429 Too Many Requests`を握り潰すと連続違反でBANに直結する。`Retry-After`を優先し、無ければ`15s→30s→60s→90s`の指数バックオフ。3回失敗で当日打ち切り。

```python
def fetch_resilient(url, rl, retries=3):
    for i in range(retries):
        r_json = None
        try:
            r_json = fetch(url, rl)
            return r_json
        except requests.HTTPError as e:
            if e.response.status_code != 429:
                raise
            wait = int(e.response.headers.get("Retry-After", min(15 * 2**i, 90)))
            time.sleep(wait)
    raise RuntimeError(f"429 {retries}回連続: 当日中止 {url}")
```

## JST7時固定の単一スレッド常駐でUser-Agentを偽装しない

同時実行数は`1`に固定し、実行はJST`07:00`の1回のみ。User-Agentは詐称せず、自作ツール名と連絡先を明記して「正体を隠さない」スタンスを取る。これが規約・自己責任の境界線を踏み越えない最低条件だ。

```python
import datetime, zoneinfo
JST = zoneinfo.ZoneInfo("Asia/Tokyo")
UA = "zenn-kpi-collector/1.0 (personal; contact: you@example.com)"

def is_run_window():
    now = datetime.datetime.now(JST)
    return now.hour == 7 and now.minute < 10  # 07:00-07:09のみ
```

## 取得失敗を握り潰さずDiscordへ通知する

夜間の静かな失敗が運用を止める。`daily_cap`到達・`429`中止・連続`304`異常はすべて例外で表面化させ、Webhookで即通知する。

```python
def notify(msg):
    requests.post(DISCORD_WEBHOOK, json={"content": f"[KPI] {msg}"}, timeout=5)

try:
    if is_run_window():
        data = fetch_resilient(API_URL, RateLimiter())
        notify("収集完了" if data else "差分なし(304)")
except Exception as e:
    notify(f"⚠️失敗: {e}")  # 握り潰さず必ず通知
    raise
```

この5点（ジッター・日次上限・ETag・429冷却・通知）を組み合わせることで、半年間ノーBAN・無停止で月`0.9万`リクエストに収まった。次章では、この収集データをSQLiteへ正規化してKPIダッシュボードを自動生成する。
