---
title: "第2章: Playwright 1.44でXミュートキーワードをCSV→UI自動同期 月400件更新を5分に圧縮"
free: false
---

## Filtered Stream API ¥14,800/月の壁とPlaywright 1.44迂回の判断基準

X APIのBasicプランはWrite権限なし。Filtered Streamへのキーワード自動登録にはEnterprise相当が必要で、2024年時点で月$100超。12週間検証した結論として**UIオートメーション迂回**が現実解になった。

Playwright 1.44はChromiumのheadlessモードで`aria-label`や`data-testid`による安定セレクタが取れ、XのSPAとも戦える。

```bash
pip install playwright==1.44.0 pyotp
playwright install chromium
```

## セレクタ3世代の崩壊履歴 — XのReact動的レンダリング対策

XはReact + Next.jsのSPA。DOMが定期的に再構築される。実際に崩壊した3パターン:

| 世代 | セレクタ例 | 崩壊時期 | 原因 |
|------|-----------|---------|------|
| v1 | `input[placeholder="キーワード"]` | 2023-11 | プレースホルダ文字列変更 |
| v2 | `div[data-testid="mute-keyword-input"]` | 2024-02 | data-testid削除 |
| v3（現行） | `[aria-label="Add muted keyword"]` | — | aria-labelは比較的安定 |

`aria-label`は言語設定で変わるため、**アカウントを英語UIに固定**することが前提。

```python
# v3セレクタでの入力確認
await page.locator('[aria-label="Add muted keyword"]').click()
await page.locator('[aria-label="Enter keyword"]').fill("副業詐欺")
await page.locator('[data-testid="confirmationSheetConfirm"]').click()
await page.wait_for_timeout(800)
```

## CSV→X差分同期ロジック全実装（Playwright 1.44）

```python
import asyncio, csv, json, os
from playwright.async_api import async_playwright

async def get_current_mutes(page) -> set[str]:
    await page.goto("https://x.com/settings/muted_keywords")
    await page.wait_for_load_state("networkidle")
    items = page.locator('[data-testid="mute-keyword-row"] span')
    return set((await items.all_inner_texts()))

async def add_keyword(page, kw: str):
    await page.locator('[aria-label="Add muted keyword"]').click()
    await page.locator('[aria-label="Enter keyword"]').fill(kw)
    await page.locator('[data-testid="confirmationSheetConfirm"]').click()
    await page.wait_for_timeout(800)

async def remove_keyword(page, kw: str):
    row = page.locator(f'[data-testid="mute-keyword-row"]:has-text("{kw}")')
    await row.locator('[aria-label="Remove"]').click()
    await page.locator('[data-testid="confirmationSheetConfirm"]').click()
    await page.wait_for_timeout(800)

async def sync_mutes(csv_path: str, cookies_path: str):
    with open(csv_path, newline="", encoding="utf-8") as f:
        target = {row[0].strip() for row in csv.reader(f) if row}

    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        ctx = await browser.new_context()
        with open(cookies_path) as f:
            await ctx.add_cookies(json.load(f))
        page = await ctx.new_page()

        current = await get_current_mutes(page)
        to_add    = target - current
        to_remove = current - target
        print(f"追加: {len(to_add)}件 / 削除: {len(to_remove)}件")

        for kw in to_add:
            await add_keyword(page, kw)
        for kw in to_remove:
            await remove_keyword(page, kw)

        await browser.close()

asyncio.run(sync_mutes("keywords.csv", "cookies.json"))
```

400件差分（追加300/削除100）の実測所要時間は**4分38秒**。`wait_for_timeout(800)`の積み上げが支配的なので、並列化より削除件数を月次にまとめる運用の方が効く。

## 2FA突破: pyotp + Playwright TOTPフロー

セッション切れ時の再ログインで2FAが入る。`pyotp`で自動入力できる:

```python
import pyotp

async def handle_2fa(page, totp_secret: str):
    await page.fill('[autocomplete="username"]', os.environ["X_USERNAME"])
    await page.click('[data-testid="LoginForm_Login_Button"]')
    await page.fill('[name="password"]', os.environ["X_PASSWORD"])
    await page.click('[data-testid="LoginForm_Login_Button"]')

    totp_locator = page.locator('[data-testid="LoginForm_Enter_2FA_code_label"]')
    if await totp_locator.is_visible(timeout=5000):
        code = pyotp.TOTP(totp_secret).now()
        await totp_locator.fill(code)
        await page.click('[data-testid="LoginForm_Login_Button"]')
```

`X_TOTP_SECRET`はGitHub Secretsに格納し、コード中にハードコードしない。

## GitHub Actions週次自動実行 — .ymlとローカル移行4ステップ

```yaml
# .github/workflows/sync-mutes.yml
name: Sync X Mute Keywords
on:
  schedule:
    - cron: "0 1 * * 1"   # 毎週月曜 10:00 JST
  workflow_dispatch:

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install playwright==1.44.0 pyotp && playwright install chromium --with-deps
      - run: echo '${{ secrets.X_COOKIES_JSON }}' > cookies.json
      - env:
          X_USERNAME:    ${{ secrets.X_USERNAME }}
          X_PASSWORD:    ${{ secrets.X_PASSWORD }}
          X_TOTP_SECRET: ${{ secrets.X_TOTP_SECRET }}
        run: python sync_mutes.py keywords.csv cookies.json
```

**ローカル→Actions移行の4ステップ:**

1. ローカルで`sync_mutes.py`を実行し`cookies.json`を取得
2. `cat cookies.json`の内容をGitHub Secretsの`X_COOKIES_JSON`に貼り付け
3. `keywords.csv`をリポジトリにコミット（機密情報がなければpublicでも可）
4. `workflow_dispatch`で手動実行し、ログで追加/削除件数が出れば完了

セッションは7〜14日で切れるため、週1回のcookies更新が運用コストの全て。完全無人化は2FAがある限り困難だが、月400件のミュート同期が5分以内に収まればこの構成で十分に回る。
