---
title: "第4章 Playwrightでミュート操作を自動化:API非対応のUIを叩く待機戦略とBAN回避"
free: false
---

## storage_stateでログインを1回だけにする

毎回ログインするとパスワード入力とOTPでBOT判定されやすい。先に手動ログインしたセッションを`x_state.json`へ書き出し、以降のミュート処理は全てそれを読み込んで起動する。半年の運用で再ログインが必要になったのは2回だけだった。

```python
# save_session.py（最初に1回だけ手動実行）
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=False)
    ctx = browser.new_context()
    page = ctx.new_page()
    page.goto("https://x.com/login")
    input("ログイン完了後にEnter> ")  # OTPも手動で通す
    ctx.storage_state(path="x_state.json")
```

## role/text基準でミュートボタンを掴む

CSSクラスは難読化されて日替わりで変わるため、`get_by_role`と表示テキストで指定する。X(旧Twitter)のミュートはユーザーメニュー内に出るので、まずメニューを開いてから「@user さんをミュート」のテキストを叩く。

```python
def mute(page, handle: str):
    page.goto(f"https://x.com/{handle}")
    page.get_by_test_id("userActions").click()  # …メニュー
    page.get_by_role("menuitem", name="ミュート").click()
```

## ミュート出現を待つ明示的待機と失敗時スクショ

`time.sleep`固定待ちはDOM描画前に空振りする。`expect`でメニュー項目の可視化を最大8秒待ち、失敗したら即スクショを残して次へ進める。これで原因不明の取りこぼしがログだけで追えるようになった。

```python
from playwright.sync_api import expect

def mute_safe(page, handle: str):
    try:
        page.goto(f"https://x.com/{handle}")
        item = page.get_by_role("menuitem", name="ミュート")
        page.get_by_test_id("userActions").click()
        expect(item).to_be_visible(timeout=8000)
        item.click()
    except Exception:
        page.screenshot(path=f"err_{handle}.png")
        raise
```

## ジッター待機と1日上限でレート制限を踏まない

連続実行で1分あたり30件叩いた日に、メニューが開かず全件スクショ落ちした。原因はソフトなレート制限だった。1操作ごとに8〜20秒のジッター、1日40件上限に下げてから再発ゼロ。

```python
import random, time

DAILY_LIMIT = 40
def run(page, handles):
    for h in handles[:DAILY_LIMIT]:
        mute_safe(page, h)
        time.sleep(random.uniform(8, 20))  # 固定値は機械的すぎる
```

```text
# err.log（実際の失敗）
12:03:11 muted @spam_a
12:03:13 muted @spam_b   ← 間隔2秒
12:03:41 TimeoutError userActions  ← 以降30件連続で空振り
```

## ヘッドレス検知を避ける最小コンテキスト設定

`headless=True`の既定UAは`HeadlessChrome`を含み弾かれる。User-Agentとロケール、ビューポートを実機並みに揃えるだけで、メニュー描画の失敗率が体感で1/5になった。過度なステルス化は不要で、この3点で足りる。

```python
ctx = browser.new_context(
    storage_state="x_state.json",
    user_agent=("Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                "AppleWebKit/537.36 (KHTML, like Gecko) "
                "Chrome/124.0.0.0 Safari/537.36"),
    locale="ja-JP",
    viewport={"width": 1280, "height": 800},
)
```

ノイズ送信元の自動ミュートはこの構成で完結する。次章では、ミュート後に残った投稿をClaude Haikuで分類し、定時Digestへ束ねる処理へつなぐ。
