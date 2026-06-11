---
title: "第2章 connect_over_cdpで実Chromeを乗っ取る — プロファイル流用とcookie永続化"
free: false
---

## --remote-debugging-port=9222でChromeを起動する

第1章で`navigator.webdriver`が`true`になりPlaywright launchが弾かれた。CDPアタッチ方式なら起動元が実Chromeになるため、このフラグは`false`のまま維持される。まず投稿用プロファイルを指定してChromeをデバッグポート付きで立ち上げる。

```bash
# Windows: 投稿専用プロファイルを9222で公開起動
& "C:\Program Files\Google\Chrome\Application\chrome.exe" `
  --remote-debugging-port=9222 `
  --user-data-dir="C:\bot\profiles\acc_a" `
  --no-first-run --no-default-browser-check
```

`--user-data-dir`を既存ログイン済みディレクトリのコピーに向ける点が肝で、cookieもlocalStorageもそのまま引き継がれる。

## connect_over_cdpでアタッチする最小コード

`launch_persistent_context`はPlaywright管理下のChromiumを起動するため検知特徴が残る。対して`connect_over_cdp`は手動起動した実Chromeへ後付けで接続するだけなので、UA・WebGLベンダ・プラグイン構成がすべて本物になる。

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.connect_over_cdp("http://localhost:9222")
    ctx = browser.contexts[0]          # 既存プロファイルのcontextを再利用
    page = ctx.pages[0] if ctx.pages else ctx.new_page()
    page.goto("https://www.pinterest.com/pin-builder/")
    print(page.evaluate("() => navigator.webdriver"))  # -> False
```

`browser.new_context()`を呼ぶと空セッションになりログインが消える。必ず`contexts[0]`を掴むこと。

## storage_stateでcookieを永続化し2段階認証を1回で終わらせる

2段階認証は初回だけ手動通過させ、その時点のcookie群をJSON化して以降は無人で再注入する。Pinterest/X/noteの3サイトとも、この方式でSMS認証の再要求が出なくなった実測。

```python
# 手動ログイン直後に1度だけ実行
ctx.storage_state(path="state/acc_a.json")

# 以降の無人投稿側ではプロファイル更新が壊れた時の保険として復元
import json, pathlib
state = json.loads(pathlib.Path("state/acc_a.json").read_text("utf-8"))
print(f"cookies={len(state['cookies'])}")  # acc_aは平均31〜44件で安定
```

プロファイル流用が主、storage_state復元が副の二重化にすると、profileロック競合で落ちた朝でも投稿継続できる。

## ポート分離で3アカウントを並行投稿する構成

アカウントごとにポートと`user-data-dir`を分け、同一PC内で並行接続する。9222/9223/9224に割り当て、深夜帯に3枚同時送信して月600枚レーンを作る。

```python
ACCOUNTS = [
    {"name": "acc_a", "port": 9222, "site": "pinterest"},
    {"name": "acc_b", "port": 9223, "site": "x"},
    {"name": "acc_c", "port": 9224, "site": "note"},
]

def attach(p, port):
    return p.connect_over_cdp(f"http://localhost:{port}").contexts[0]

with sync_playwright() as p:
    sessions = {a["name"]: attach(p, a["port"]) for a in ACCOUNTS}
```

## BAN回避: 投稿間隔ジッタとプロファイル分離ルール

3アカウントを同一IP・同一秒で投げると相関検知される。投稿間に乱数ジッタを挟み、`user-data-dir`は絶対に共有しない。共有した翌日にacc_bが7日サスペンドを受けたのが分離を固定した理由。

```python
import random, time

def jitter_sleep(base=90, spread=70):
    # 90±70秒。3サイトの最短投稿間隔から逆算した安全域
    time.sleep(base + random.uniform(-spread, spread))

for acc in ACCOUNTS:
    post_one(sessions[acc["name"]], acc["site"])
    jitter_sleep()
```

ポート分離・プロファイル分離・ジッタの3点を守る限り、CDP接続側がBAN要因になった事例は3サイト90日でゼロだった。
