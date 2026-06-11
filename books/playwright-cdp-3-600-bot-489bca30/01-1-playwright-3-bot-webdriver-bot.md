---
title: "第1章 なぜ素のPlaywrightは3分でBotバレするか — webdriver検知と完成Botの全体像"
free: true
---

本章を読み終えると、`navigator.webdriver` が `true` を返す瞬間を自分の目で確認し、本書最終形のBot — 深夜2時にcron起動して3サイトへ計600枚/月を連続投稿し、結果をDiscordへ通知する構成 — の全体図を手元に持てる。

結論を先に出す。素のPlaywrightは `p.chromium.launch()` の起動直後に弾かれる。原因は2つ、`navigator.webdriver=true` とCDP由来の痕跡。対策は「新規起動をやめ、既存Chromeに `connect_over_cdp` で繋ぐ」こと。これが3サイト共通の土台になる。

## launch() 直後に navigator.webdriver が true で即弾きされる検証コード

まず再現する。Bot検知サイト `bot.sannysoft.com` を素のPlaywrightで開き、`webdriver` 行が赤くなる様子をスクショで残す。

```python
# detect_check.py
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=False)
    page = browser.new_page()
    page.goto("https://bot.sannysoft.com/")
    print("webdriver =", page.evaluate("() => navigator.webdriver"))  # -> True
    page.screenshot(path="sannysoft_launch.png", full_page=True)
    browser.close()
```

`webdriver = True` が出力され、スクショの該当行は赤。3サイトのうち2サイトはこの時点で投稿フォームに到達できず、平均3分以内にセッションが切られた（実測ログは第4章）。

## connect_over_cdp で既存Chromeプロファイルに繋ぎ検知を抜く

`launch()` を捨てる。`--remote-debugging-port=9222` で起動済みの普段使いChromeへ後付けで接続すると、`webdriver` は `false` を返す。

```bash
# 既存プロファイルのままデバッグポートで起動 (Windows / PowerShell)
& "C:\Program Files\Google\Chrome\Application\chrome.exe" `
  --remote-debugging-port=9222 `
  --user-data-dir="C:\Users\ityya\chrome-bot-profile"
```

```python
# connect_cdp.py
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp("http://localhost:9222")
    page = browser.contexts[0].pages[0]
    page.goto("https://bot.sannysoft.com/")
    print("webdriver =", page.evaluate("() => navigator.webdriver"))  # -> False
```

同じ検知サイトで全行が緑に変わる。ログイン済みCookieもプロファイルごと流用できるため、毎回のログイン認証も消える。

## 深夜2時cron→3サイト連続投稿→Discord通知の到達点

本書のゴールは次のディレクトリ構成で動く。各サイトの差分は `sites/` 配下に隔離する。

```
night-poster/
├── docker-compose.yml      # chrome(CDP) + poster の2コンテナ
├── poster/
│   ├── main.py             # cron 2:00 起動エントリ
│   ├── cdp.py              # connect_over_cdp ラッパ
│   ├── notify_discord.py   # Webhook通知
│   └── sites/
│       ├── site_a.py       # フォーム差分A
│       ├── site_b.py       # 差分B
│       └── site_c.py       # 差分C
└── .env                    # DISCORD_WEBHOOK_URL 等
```

```python
# poster/main.py の骨格
from sites import site_a, site_b, site_c
from notify_discord import notify

posted = 0
for site in (site_a, site_b, site_c):
    posted += site.run()          # 各サイト 200枚/月 = 計600枚
notify(f"深夜投稿完了: {posted}枚")
```

## docker compose up 一発で起動する到達構成

最終操作はコマンド1行に集約する。CDP付きChromeとposterが同時に立ち上がる。

```yaml
# docker-compose.yml (抜粋)
services:
  chrome:
    image: zenika/alpine-chrome:124
    command: ["--no-sandbox", "--remote-debugging-address=0.0.0.0",
              "--remote-debugging-port=9222",
              "--user-data-dir=/profile"]
    volumes: ["./profile:/profile"]
  poster:
    build: ./poster
    depends_on: [chrome]
    environment:
      - CDP_URL=http://chrome:9222
```

```bash
docker compose up -d
docker compose logs -f poster   # "深夜投稿完了: 600枚" を待つ
```

## 第2章以降で埋める3つの壁

土台はできたが、これだけでは1週間でBANされる。残る壁は3つ。

```text
壁1: フォーム差分   3サイトでセレクタ/アップロードUIが全部違う        → 第2章
壁2: 遅延設計       0.3秒間隔の連投は機械的すぎてレート制限に当たる    → 第3章
壁3: BAN回避運用    1アカウント月200枚の上限・投稿間隔・UA固定の実測   → 第4章
```

```python
# 壁2の予告: 人間らしい遅延 (詳細は第3章)
import random, time
def human_sleep(lo=4.0, hi=11.0):
    time.sleep(random.uniform(lo, hi))   # 固定sleepはBANフラグ
```

`connect_over_cdp` で検知を抜けても、3サイト分のフォーム差分・遅延・BAN回避を詰めなければ月600枚は安定しない。次章では最初の壁「フォーム差分」を、3サイトの実セレクタとアップロード処理の差分込みで埋めていく。
