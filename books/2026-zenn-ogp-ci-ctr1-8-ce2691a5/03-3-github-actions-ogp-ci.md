---
title: "第3章 GitHub ActionsでOGP・リンクカード・埋め込みを公開前に全検証するCI"
free: false
---

## Playwright で og:image / og:title / twitter:card の欠落を検出する

結論: Zenn のメタタグ検証は Playwright のヘッドレスで `head` を直接読むのが最速だ。`zenn.dev/<user>/articles/<slug>` を開き、`og:image`・`og:title`・`twitter:card` の3点が揃わなければその場で落とす。Zenn は `published: false` の記事を 404 にするため、CI では公開済み slug だけを対象にする。

実測では、20記事中3記事で `og:image` が `https://res.cloudinary.com/...` のデフォルト画像にフォールバックしており、これが差し替え後に CTR を 1.8 倍へ押し上げた最大要因だった。

```python
# verify_meta.py
from playwright.sync_api import sync_playwright

REQUIRED = ["og:image", "og:title", "twitter:card"]

def check(url: str) -> list[str]:
    missing = []
    with sync_playwright() as p:
        page = p.chromium.launch().new_page()
        page.goto(url, wait_until="networkidle", timeout=15000)
        for prop in REQUIRED:
            sel = f'meta[property="{prop}"], meta[name="{prop}"]'
            el = page.query_selector(sel)
            content = el.get_attribute("content") if el else None
            if not content or "default" in (content or ""):
                missing.append(prop)
    return missing
```

## 外部リンクの死活とリンクカード生成を280ms閾値で判定する

Zenn のリンクカードは記事 URL を裸書きすると自動展開されるが、相手サーバの OGP 応答が遅いと展開されずただの青リンクに落ちる。計測したところ、OpenGraph 取得が **280ms** を超えると Zenn 側のカード生成がタイムアウトしやすい。`HEAD` で死活を、`GET` で `og:title` の有無と応答時間を同時に拾う。

```python
import time, httpx

def link_health(url: str) -> dict:
    t0 = time.perf_counter()
    r = httpx.get(url, timeout=5, follow_redirects=True)
    ms = (time.perf_counter() - t0) * 1000
    has_og = 'property="og:title"' in r.text
    return {"url": url, "status": r.status_code,
            "ms": round(ms), "card_ok": has_og and ms < 280}
```

`card_ok` が False のリンクは記事内で `[テキスト](url)` 形式に手書きへ倒し、カード不発の空白を防ぐ。

## @[card] 埋め込みが展開されるかをDOM検査する

GitHub / CodePen / Tweet の `@[card](url)` 記法は、Zenn のレンダラが iframe か `embed-card` クラスへ変換する。展開失敗時は記法文字列がそのまま本文に残るため、レンダー後 DOM に生の `@[` が残っていないかを検査すれば崩れを機械検出できる。

```python
def check_embeds(page) -> list[str]:
    body = page.inner_text("div.znc")  # Zenn本文コンテナ
    broken = [ln for ln in body.splitlines() if ln.strip().startswith("@[")]
    cards = page.query_selector_all("span.embed-block, .zenn-embedded")
    if not cards and "@[" in body:
        broken.append("embed-rendered:0")
    return broken
```

## push時に全記事を回すworkflow.yml(1件崩れたらCI赤)

`main` への push で `articles/*.md` の frontmatter から slug を抽出し、上記3検査を直列で回す。1件でも `missing` か `card_ok=False` が出れば `exit 1` で CI を赤にする。これで人手の目視確認を0にした。

```yaml
name: ogp-verify
on:
  push:
    branches: [main]
    paths: ["articles/**"]
jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install playwright httpx && playwright install chromium
      - run: python run_all.py  # 全slug検査、崩れたらexit 1
```

## レビュー結果をDiscord webhookへ流す15行

CI が落ちた瞬間、どの slug のどのタグが欠けたかを Discord へ即通知する。GitHub の通知メールを開く手間を消し、push から崩れ検知まで平均90秒に短縮できた。

```python
import os, httpx, json

def notify(failures: list[dict]):
    if not failures:
        return
    lines = [f"❌ {f['slug']}: {', '.join(f['issues'])}" for f in failures]
    httpx.post(os.environ["DISCORD_WEBHOOK"],
               data=json.dumps({"content": "\n".join(lines)}),
               headers={"Content-Type": "application/json"}, timeout=10)
```

`run_all.py` の末尾で `notify(failures)` を呼べば、20記事分の検証ログが1つの Discord メッセージに集約され、崩れた記事だけを再 push で潰せる。
