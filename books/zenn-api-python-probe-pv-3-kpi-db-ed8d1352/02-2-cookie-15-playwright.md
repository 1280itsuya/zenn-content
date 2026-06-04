---
title: "第2章: Cookie認証を15分で自動取得 — Playwrightでセッション失効を検知する"
free: false
---

## connect.sid と __Host- 系Cookieだけを抜く — 全Cookie保存は失効検知を壊す

Zenn統計APIの認証実体は `connect.sid` 1個だ。Playwrightで全Cookieを `storage_state.json` に保存すると、CDNや解析タグの短命Cookieが混ざり、再利用時に「どれが切れたのか」を切り分けられなくなる。必要なのは `connect.sid` と `__Host-` プレフィックスのセッション系のみに絞ることだ。

```python
from playwright.sync_api import sync_playwright

KEEP = {"connect.sid"}

def login_and_save(email: str, password: str, path="storage_state.json"):
    with sync_playwright() as p:
        b = p.chromium.launch(headless=True)
        ctx = b.new_context()
        page = ctx.new_page()
        page.goto("https://zenn.dev/enter")
        page.fill('input[type="email"]', email)
        page.fill('input[type="password"]', password)
        page.click('button[type="submit"]')
        page.wait_for_url("https://zenn.dev/dashboard", timeout=15000)
        state = ctx.storage_state()
        state["cookies"] = [c for c in state["cookies"]
                            if c["name"] in KEEP or c["name"].startswith("__Host-")]
        import json; json.dump(state, open(path, "w"))
        b.close()
```

## 401と302を別物として扱う — Zennは失効を200+HTMLで返す

最大の罠は、Zennが認証失効時に素直な401を返さないことだ。`/api/me` 系へ未認証で叩くと302で `/enter` へ飛び、最終的に**ステータス200のログインHTML**が返る。`resp.ok` だけ見るとTrueになり、PV取得は静かにゼロを記録する。

```python
import httpx, json

def load_state(path="storage_state.json"):
    cookies = {c["name"]: c["value"] for c in json.load(open(path))["cookies"]}
    return cookies

def fetch_stats(slug: str, cookies: dict):
    r = httpx.get(f"https://zenn.dev/api/articles/{slug}/stats",
                  cookies=cookies, follow_redirects=False)
    if r.status_code in (301, 302) and "/enter" in r.headers.get("location", ""):
        raise AuthExpiredError(f"302 redirect to login: slug={slug}")
    return r
```

## Content-Typeで殴る — JSONを期待してHTMLが来たら即raise

302を踏まず200+HTMLで返るケースも `follow_redirects=True` だと発生する。レスポンスが `text/html` なら問答無用で `AuthExpiredError` を投げ、probe全体を落とす。「落として気づく」のが本書の設計思想だ。

```python
class AuthExpiredError(RuntimeError):
    pass

def parse_pv(resp: httpx.Response) -> int:
    ctype = resp.headers.get("content-type", "")
    if "application/json" not in ctype:
        raise AuthExpiredError(f"expected JSON, got {ctype!r} ({len(resp.text)}B)")
    return resp.json()["page_view"]
```

`got 'text/html; charset=utf-8' (48213B)` のような例外メッセージが、後述のSentry/Discord通知でそのまま原因特定の手掛かりになる。

## 失効2週間で欠測した実損 — 14日 × 平均1,820PV/日が記録漏れ

筆者は2026年3月、`connect.sid` の失効に14日間気づかなかった。KPI DBには毎朝7時のcronが `page_view=0` を14行INSERTし続け、グラフ上は「全記事が突然死んだ」ように見えた。実際は取得失敗で、復旧後に判明した欠測は累計 **25,480 PV**(平均1,820PV/日 × 14日)。`AuthExpiredError` を導入していれば初日の7:00時点でcronが非ゼロ終了し、Discordに飛んでいた損失だ。

```python
def main():
    cookies = load_state()
    try:
        for slug in target_slugs():
            pv = parse_pv(fetch_stats_followed(slug, cookies))
            db_upsert(slug, pv)            # 0は絶対に書かない
    except AuthExpiredError as e:
        notify_discord(f"⚠ Zenn auth expired: {e}")
        raise SystemExit(2)               # cron が異常終了として検知
```

## storage_state.jsonの自己回復 — 失効検知から再ログインまで45秒

失効を検知したら、人手を介さず `login_and_save()` を再実行して `storage_state.json` を上書きし、同一プロセス内でprobeをリトライする。headlessログインから再取得までの実測は約45秒。これで「2週間欠測」は「1サイクル遅延」に縮む。

```python
def run_with_recovery():
    try:
        main()
    except SystemExit as e:
        if e.code != 2:
            raise
        login_and_save(os.environ["ZENN_EMAIL"], os.environ["ZENN_PASSWORD"])
        main()  # 新しい connect.sid で再実行
```

`__Host-` 系を捨てずに残したのは、Zennが将来 `connect.sid` をCSRFトークンと二段化したときに、`KEEP` 集合を1行足すだけで追従するためだ。スキーマ変更を「例外で気づく」設計が、SDKなし運用を2年支えている。

---

**topics（Zenn Book 公開用タグ）:** `claude` / `python` / `sqlite` / `automation` / `zenn`
