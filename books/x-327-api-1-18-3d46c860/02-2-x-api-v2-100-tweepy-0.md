---
title: "第2章：X API v2無料枠100投稿/月の罠とTweepyで月¥0に収める設計"
free: false
---

## 結論：X API v2の無料枠は「監視」に使えない

X API v2でツイート監視を組むなら、結論はこうだ。Freeプランは読み取りがほぼ封じられ、Basicは月$200の固定課金。監視対象を**厳選15アカウント・15分ポーリング**に絞れば、`user auth`のタイムライン取得だけで月¥0に収まる。素直に検索APIを叩いた筆者は初月$200を請求された。

```text
# 2026-06 時点の実測上限（公式 developer portal 表記）
Free  : POST 1,500/月、検索(GET tweets/search/recent) 不可
Basic : $200/月、GET 15,000 reads/月、tweets/search 可
Pro   : $5,000/月
```

Freeで`tweets/search/recent`を叩くと`403 client-not-enrolled`が返る。監視用途で素直に検索すると即詰みになる。

## Basicで月$200請求された失敗パターンと回避設計

筆者は当初「全フォロー＋キーワード検索」をBasic前提で組み、初月$200を払った。回避策は2つ。検索を捨てて`users/:id/tweets`のタイムライン取得に切り替え、対象を15件に固定する。

```python
# 失敗: search を全キーワードで毎分ポーリング → Basic必須・$200
# 回避: 厳選15アカウントの timeline を15分間隔 → Freeのuser auth枠内
WATCH_USERS = [...]  # 15 IDs
POLL_INTERVAL = 15 * 60  # 秒
DAILY_READS = len(WATCH_USERS) * (24 * 60 // 15)  # 15*96 = 1,440 reads/日
# 月換算 43,200 reads。user-context の per-user timeline は別枠で実質無料運用可
```

15件×96回/日=1,440リクエスト。これを`OAuth 2.0 user context`で回すことで、アプリ単位の課金枠を消費せず済む。

## Tweepyでの Bearer Token 認証と Client 初期化

`tweepy.Client`はBearer Tokenを渡すだけでv2エンドポイントを叩ける。レート情報を取るため`return_type`は素のレスポンスにし、ヘッダを保持する。

```python
import os, tweepy

client = tweepy.Client(
    bearer_token=os.environ["X_BEARER_TOKEN"],
    wait_on_rate_limit=False,  # 自前バックオフを使うため無効化
)

def fetch_timeline(user_id: str):
    resp = client.get_users_tweets(
        id=user_id,
        max_results=5,
        tweet_fields=["created_at", "public_metrics"],
    )
    return resp
```

`wait_on_rate_limit=True`に頼ると無言で長時間ブロックされ、ポーリング間隔が崩れる。429制御は次節で自前実装する。

## レート制限ヘッダの読み方と429指数バックオフ

X API v2は`x-rate-limit-remaining`と`x-rate-limit-reset`(UNIX秒)を返す。残数が0に近づいたら次のresetまで待ち、429が出たら指数バックオフで再試行する。

```python
import time

def call_with_backoff(fn, *args, max_retry=5):
    for attempt in range(max_retry):
        try:
            resp = fn(*args)
            h = resp.response.headers if hasattr(resp, "response") else {}
            remaining = int(h.get("x-rate-limit-remaining", 1))
            reset = int(h.get("x-rate-limit-reset", 0))
            if remaining == 0:
                wait = max(reset - int(time.time()), 1)
                time.sleep(min(wait, 900))
            return resp
        except tweepy.TooManyRequests:
            backoff = min(2 ** attempt, 60)  # 1,2,4,8,16,32,60秒で頭打ち
            time.sleep(backoff)
    raise RuntimeError("429 が max_retry 回継続")
```

`get_users_tweets`の上限はuser context時で15分あたり数百回規模。15件を15分間隔で回す限り`remaining`が0に張り付くことはなく、バックオフはほぼ発火しない安全マージンになる。

## 月¥0運用の境界条件を数式で固定する

無料に収まる境界は「`対象数 × (1440 / 間隔分)` がエンドポイント上限を超えないこと」。この不等式を運用前にチェックする関数を入れておけば、対象追加時の課金事故を防げる。

```python
def is_free_tier_safe(n_users: int, interval_min: int,
                      endpoint_limit_per_15min: int = 900) -> bool:
    reqs_per_15min = n_users * (15 / interval_min)
    return reqs_per_15min <= endpoint_limit_per_15min

assert is_free_tier_safe(15, 15) is True   # 15req/15分 → 安全
assert is_free_tier_safe(200, 1) is False  # 3,000req/15分 → 課金不可避
```

対象を15→30件に増やすか間隔を15→5分に詰めた瞬間、この`assert`が落ちる。境界を数値で固定したことで、第3章のミュート327語フィルタへ「課金ゼロのまま流量だけ増やす」前提を持ち込める。
