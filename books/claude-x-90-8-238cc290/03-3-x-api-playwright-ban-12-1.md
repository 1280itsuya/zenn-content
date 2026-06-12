---
title: "第3章 X API無料枠の壁とPlaywright巡回：BAN率を月12%→1%に下げたヘッドレス設定"
free: false
---

## X API無料枠は月100読取で枯れる

X APIの無料プランは2025年時点で書込中心、読取は実質月100件・15分あたり1リクエストまで。3アカウントのTL監視には全く足りない。まず枯渇を実測ログで可視化する。

```python
import os, requests, datetime as dt

def read_quota_left() -> int:
    r = requests.get(
        "https://api.twitter.com/2/users/me",
        headers={"Authorization": f"Bearer {os.environ['X_BEARER']}"},
    )
    # x-rate-limit-remaining はエンドポイント毎に返る
    left = int(r.headers.get("x-rate-limit-remaining", 0))
    print(dt.datetime.now().isoformat(), "remaining=", left)
    return left
```

半年の運用ログでは、無料枠は朝のバッチ1回（30件取得）で**3日目に0**へ落ちた。Basic（月$200）を払う前に取得層を二重化する。

## API→Playwrightへ自動フォールバック

無料枠が尽きたら例外を投げず、ヘッドレス巡回へ切り替える。判定は残量0または429。

```python
def fetch_timeline(handle: str) -> list[dict]:
    try:
        if read_quota_left() <= 1:
            raise RuntimeError("quota_exhausted")
        return fetch_via_api(handle)        # API経路
    except RuntimeError:
        return fetch_via_playwright(handle)  # 巡回経路へ
```

この1関数で、API取得88%・Playwright取得12%の比率に落ち着き、$200を払わず月3,000ポスト相当を回収できた。

## BAN率12%の原因は同一IP・等間隔・UA欠落

最初の3アカウント運用で**月12%が凍結**。ログを突き合わせると共通点は3つ——固定IP、60秒ちょうどの等間隔、user-agent未設定だった。再現条件をそのまま設定で潰す。

```python
from playwright.sync_api import sync_playwright
import random, time

UA = ("Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
      "AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0 Safari/537.36")

def new_context(p):
    browser = p.chromium.launch(
        headless=True,
        proxy={"server": os.environ["RESIDENTIAL_PROXY"]},  # 住宅プロキシ
    )
    return browser.new_context(
        user_agent=UA,
        locale="ja-JP",
        viewport={"width": 1280, "height": 800},
        storage_state="state.json" if os.path.exists("state.json") else None,
    )
```

住宅プロキシ＋UA固定だけで凍結は12%→**4%**へ。残りはアクセス間隔で削る。

## ランダム遅延7〜23秒で1%まで下げる

等間隔は機械検知の主因。スクロール毎に7〜23秒の一様乱数を挟み、1セッションの取得上限も120件で打ち切る。

```python
def scroll_collect(page, limit=120) -> list[str]:
    seen, ids = set(), []
    while len(ids) < limit:
        for art in page.query_selector_all('article[data-testid="tweet"]'):
            tid = art.get_attribute("aria-labelledby") or art.inner_text()[:40]
            if tid not in seen:          # 無限スクロールの重複排除
                seen.add(tid); ids.append(tid)
        page.mouse.wheel(0, 2400)
        time.sleep(random.uniform(7, 23))  # 等間隔を崩す
    return ids
```

`seen` セットで差分だけ拾うため、同じポストを二度処理しない。この遅延導入後、半年で凍結は**1%（3アカウント×6ヶ月で実質1回）**に収束した。

## storage_stateでログイン壁を1回だけ突破

毎回ログインするとチャレンジ画面でBANが跳ねる。初回だけ手動ログインし、cookieを `state.json` に永続化して再利用する。

```python
def save_login_once():
    with sync_playwright() as p:
        ctx = p.chromium.launch(headless=False).new_context()
        page = ctx.new_page()
        page.goto("https://x.com/login")
        input("ログイン完了後にEnter > ")  # 初回のみ手動
        ctx.storage_state(path="state.json")  # 以降は無人で再利用
```

保存後は `new_context` が `state.json` を読み、ログインフローを完全に飛ばす。これでチャレンジ遭遇率がほぼ0になり、巡回経路でも無人運用が成立する。
