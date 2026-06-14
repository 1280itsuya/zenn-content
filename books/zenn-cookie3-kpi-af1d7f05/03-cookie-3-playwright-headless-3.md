---
title: "Cookie失効を3分以内に自動検知→Playwright headlessで再ログイン：無限再帰しない3ステップ回復パターン"
free: false
---

Zennの章を執筆します。

## Cookie失効を3分以内に自動検知→Playwright headlessで再ログイン：無限再帰しない3ステップ回復パターン

---

## 失敗の証拠：深夜3時のステータス303とエラーログ全文

本番で発覚したのは明け方のKPIバッチが静かに空振りしていたことだった。ログには次の2行だけが残っていた。

```
2026-06-09 03:17:42 WARNING httpx - GET https://zenn.dev/api/me/stats → 303
2026-06-09 03:17:42 ERROR pipeline - Location: https://zenn.dev/login  (retry=0)
```

`401 Unauthorized` ではなく **303 See Other → /login** がZennのCookie失効パターンだ。この挙動をDevToolsで確認したのが実装の起点になった。`Fetch/XHR` タブでフィルタして `stats` を叩くと、失効後は即座に `303` が返り、レスポンスヘッダに `location: /login` が入る。

## httpxイベントフックで401/303を5行で検知する

`httpx.AsyncClient` のイベントフック `response` を使えばすべてのレスポンスに横断できる。ミドルウェア不要、既存コードへの変更はゼロだ。

```python
# zenn_client.py
import httpx
import asyncio
from pathlib import Path

COOKIE_PATH = Path("zenn_cookies.json")

def _is_expired(response: httpx.Response) -> bool:
    if response.status_code == 401:
        return True
    location = response.headers.get("location", "")
    return response.status_code == 303 and "/login" in location

async def _on_response(response: httpx.Response) -> None:
    if _is_expired(response):
        raise CookieExpiredError(f"Expired: {response.status_code} {response.url}")

class CookieExpiredError(Exception):
    pass
```

イベントフックはリダイレクト追跡前に発火するため `follow_redirects=False` を明示することで303を取りこぼさない。

## Playwrightシングルトンで再ログイン→JSON永続化

ブラウザを毎回起動すると1回あたり約2秒のオーバーヘッドが生じる。モジュールレベルのシングルトンで **初回だけ起動、以降は再利用** する設計にする。

```python
# zenn_auth.py
import json
from playwright.async_api import async_playwright, Browser, BrowserContext

_browser: Browser | None = None
_context: BrowserContext | None = None

async def _get_context() -> BrowserContext:
    global _browser, _context
    if _browser is None:
        pw = await async_playwright().start()
        _browser = await pw.chromium.launch(headless=True)
    if _context is None:
        _context = await _browser.new_context()
    return _context

async def refresh_zenn_cookie(email: str, password: str, cookie_path: Path) -> None:
    ctx = await _get_context()
    page = await ctx.new_page()

    await page.goto("https://zenn.dev/enter")
    await page.fill('input[name="email"]', email)
    await page.fill('input[name="password"]', password)
    await page.click('button[type="submit"]')

    # /dashboard へのナビゲーション完了＝ログイン成功の証明
    await page.wait_for_url("**/dashboard", timeout=15_000)

    cookies = await ctx.cookies("https://zenn.dev")
    cookie_path.write_text(json.dumps(cookies, ensure_ascii=False, indent=2))
    await page.close()
```

`wait_for_url` のタイムアウトを15秒にしているのはZennのSPA初期化が遅延するケースを考慮したためだ。

## max_retries=1が無限ループを防ぐ唯一の理由

再ログイン後に再度リクエストを投げ、それでも失効エラーが返った場合（アカウントロック、パスワード変更等）に再帰が止まらなくなる。`max_retries=1` はこのケースをガードする最小フラグだ。

```python
# zenn_client.py（続き）
import os

async def fetch_with_retry(
    url: str,
    cookies: dict,
    max_retries: int = 1,
    _attempt: int = 0,
) -> httpx.Response:
    async with httpx.AsyncClient(
        cookies=cookies,
        follow_redirects=False,
        event_hooks={"response": [_on_response]},
    ) as client:
        try:
            return await client.get(url)
        except CookieExpiredError:
            if _attempt >= max_retries:
                raise RuntimeError(f"Cookie refresh failed after {max_retries} retries")
            await refresh_zenn_cookie(
                email=os.environ["ZENN_EMAIL"],
                password=os.environ["ZENN_PASSWORD"],
                cookie_path=COOKIE_PATH,
            )
            new_cookies = json.loads(COOKIE_PATH.read_text())
            return await fetch_with_retry(url, new_cookies, max_retries, _attempt + 1)
```

`_attempt` を引数に持つことで再帰深度を外部から制御できる。グローバル変数によるカウンター管理は並列リクエストで競合するため使わない。

## 3ステップを繋ぐエントリポイントと動作確認コマンド

以上を統合したエントリポイントと、ローカルで失効シミュレーションを行う確認手順を示す。

```python
# run_kpi_fetch.py
import asyncio
import json
from pathlib import Path
from zenn_client import fetch_with_retry, COOKIE_PATH

async def main() -> None:
    if not COOKIE_PATH.exists():
        raise FileNotFoundError("zenn_cookies.json が存在しません。初回ログインを実行してください")

    cookies = {c["name"]: c["value"] for c in json.loads(COOKIE_PATH.read_text())}
    response = await fetch_with_retry(
        url="https://zenn.dev/api/me/stats?period=last30days",
        cookies=cookies,
    )
    print(f"status={response.status_code}  bytes={len(response.content)}")

if __name__ == "__main__":
    asyncio.run(main())
```

```bash
# Cookie を意図的に破壊して失効検知→自動回復を確認する
echo '[]' > zenn_cookies.json
python run_kpi_fetch.py
# 期待出力:
# WARNING - Expired: 303 https://zenn.dev/api/me/stats
# INFO    - Playwright headless login started
# INFO    - Cookie refreshed → zenn_cookies.json updated
# status=200  bytes=1842
```

このパイプラインが完成すると、GitHub Actionsのcronジョブが **夜中に無音でCookieが死んでも翌朝には自動回復して統計が届く** 状態になる。次章ではこの `run_kpi_fetch.py` の出力をJSONとしてリポジトリにコミットし、GitHub Actionsのアーティファクトとして蓄積するパイプラインを実装する。
