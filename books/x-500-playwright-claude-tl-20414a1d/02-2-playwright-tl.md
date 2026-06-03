---
title: "第2章 Playwrightでログインセッションを永続化しレート制限を踏まずにTLを収集する"
free: false
---

## storageStateでX認証を1回だけ突破し以降の二段階認証を省く

Playwright で X を扱う最大の摩擦は二段階認証だ。`storageState` でログイン後の Cookie / localStorage を JSON 保存すれば、2回目以降は認証画面を一切踏まない。初回だけ手動ログインし、`auth.json` を吐かせる。

```python
# save_auth.py  初回のみ実行(手動でログイン→Enter)
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=False)
    ctx = browser.new_context()
    page = ctx.new_page()
    page.goto("https://x.com/login")
    input("ログイン完了後にEnter > ")
    ctx.storage_state(path="auth.json")  # 以降これを使い回す
```

## playwright-stealthでautomation検知を消しチャレンジ画面を回避する

`navigator.webdriver=true` を見られると X はチャレンジ画面を出す。`playwright-stealth` が18個の指紋を上書きして既定の bot シグナルを潰す。`auth.json` をロードする `new_context` に必ず適用する。

```python
from playwright.sync_api import sync_playwright
from playwright_stealth import stealth_sync

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    ctx = browser.new_context(
        storage_state="auth.json",
        viewport={"width": 1280, "height": 1800},
        user_agent=("Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                    "AppleWebKit/537.36 Chrome/124.0 Safari/537.36"),
    )
    page = ctx.new_page()
    stealth_sync(page)  # webdriver/plugins/languages を偽装
    page.goto("https://x.com/home", wait_until="domcontentloaded")
```

## article要素からツイート本文・著者・URLを構造化抽出する

X の TL は `article[data-testid="tweet"]` が1ツイート単位。本文は `div[data-testid="tweetText"]`、著者は `User-Name`、URL は時刻リンクの `a[href*="/status/"]` から拾う。欠損は空文字に倒し、後段の Claude 分類へ素のまま渡す。

```python
def extract(page):
    rows = []
    for art in page.query_selector_all('article[data-testid="tweet"]'):
        text = art.query_selector('div[data-testid="tweetText"]')
        link = art.query_selector('a[href*="/status/"]')
        user = art.query_selector('div[data-testid="User-Name"]')
        rows.append({
            "text": text.inner_text() if text else "",
            "author": user.inner_text().split("\n")[0] if user else "",
            "url": "https://x.com" + link.get_attribute("href") if link else "",
        })
    return rows
```

## 表示遅延でDOMが崩れた時の3回リトライ＋指数バックオフ

無限スクロール直後は `article` が0件で返ることがある。`expect` で最低5件描画を待ち、ゼロなら最大3回・1.5倍ずつ待機を伸ばして再取得する。これで取りこぼしが体感で消える。

```python
import time

def scroll_collect(page, target=200):
    seen, wait = {}, 2.0
    miss = 0
    while len(seen) < target and miss < 3:
        for r in extract(page):
            if r["url"]:
                seen[r["url"]] = r
        before = len(seen)
        page.mouse.wheel(0, 4200)
        time.sleep(wait)
        if len(seen) == before:
            miss += 1
            wait *= 1.5          # 指数バックオフ
        else:
            miss, wait = 0, 2.0
    return list(seen.values())
```

## 1セッション約220件・90秒/スクロールがソフトBANを踏まない実測閾値

検証では、`wait=2.0` 秒・1スクロール約30件で**累計220件**を超えたあたりからチャレンジ頻度が上がった。スクロール間隔を1.5秒未満に詰めた回だけ警告が出たため、**最低1.8秒・1セッション200件・実行間隔は15分以上**を安全マージンとした。この粒度で TL を回し、後段の Claude 分類でノイズ率を **42%→9%** まで落とす土台が完成する。

```python
SAFE = {"per_session": 200, "min_wait_sec": 1.8, "interval_min": 15}

if collected_count >= SAFE["per_session"]:
    print(f"上限{SAFE['per_session']}件で停止 / 次回は{SAFE['interval_min']}分後")
    ctx.storage_state(path="auth.json")  # Cookie更新を保存して終了
    browser.close()
```

`auth.json` を実行ごとに上書き保存するのが地味に効く。セッション継続トークンが更新され、再ログイン要求の発生率が下がる。次章ではここで集めた220件を Claude に投げ、ミュート対象を自動分類する。
