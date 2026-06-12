---
title: "Civitai API+Playwrightで学習画像300枚を25分で自動収集する"
free: false
---

```yaml
# book/config.yaml に設定する公開必須フィールド
topics: ["stablediffusion", "lora", "python", "playwright", "ai"]
```

LoRA量産の律速は学習ではなく収集にある。RTX 4090で学習自体は18分、対して手動収集は1キャラ3時間。この章では Civitai API + Playwright のハイブリッド構成で300枚を25分に短縮し、収集失敗率を初版の31%から4%に下げた実装をそのまま渡す。

## Civitai /api/v1/images でリアクション200超だけを引く

Civitai の公開 REST API はタグ・NSFWレベル・リアクション数でフィルタできる。後段の数値スコア選定(第3章)の前段として、ここで「heartCount + likeCount >= 200」の足切りを入れるだけで候補が約1/8に減り、ダウンロード量が激減する。

```python
import requests, time

BASE = "https://civitai.com/api/v1/images"

def fetch_candidates(tag: str, min_reactions: int = 200, want: int = 300):
    items, cursor = [], None
    while len(items) < want:
        params = {"limit": 100, "sort": "Most Reactions",
                  "nsfw": "None", "tags": tag}
        if cursor:
            params["cursor"] = cursor
        r = request_with_backoff(BASE, params)
        data = r.json()
        for it in data["items"]:
            s = it["stats"]
            if s["heartCount"] + s["likeCount"] >= min_reactions:
                items.append({"id": it["id"], "url": it["url"],
                              "width": it["width"], "meta": it.get("meta")})
        cursor = data["metadata"].get("nextCursor")
        if not cursor:
            break
        time.sleep(1.2)  # 平常時レート: 1.2秒間隔で429ほぼゼロ
    return items[:want]
```

## 429は指数バックオフで処理する——失敗率31%→4%の正体

初版は `time.sleep(1)` 固定で、300枚中93枚(31%)が429かタイムアウトで欠落した。原因は Civitai 側のバースト制限が「直近60秒の総量」で効くため、固定スリープでは回復を待てないこと。指数バックオフ+最大5リトライに変えただけで欠落は12枚(4%)になった。

```python
def request_with_backoff(url, params=None, max_retry=5):
    for attempt in range(max_retry):
        r = requests.get(url, params=params, timeout=30)
        if r.status_code == 200:
            return r
        if r.status_code == 429:
            wait = 2 ** attempt * 3  # 3,6,12,24,48秒
            time.sleep(wait)
            continue
        r.raise_for_status()
    raise RuntimeError(f"429 persisted after {max_retry} retries: {url}")
```

## APIが返さない原寸はPlaywrightでフォールバック取得

API の `url` は幅450pxのCDNサムネイルを返すケースがある。URL中の `width=450` を原寸指定に書き換えても解決しない画像(全体の約7%)だけ、Playwright で画像詳細ページを開き原本URLを抜く。300枚中フォールバックに回るのは20枚前後なので、ヘッドレスChromiumの起動コストは1回で済ませる。

```python
from playwright.sync_api import sync_playwright

def resolve_fullres(image_ids: list[int]) -> dict[int, str]:
    results = {}
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        page = browser.new_page()
        for iid in image_ids:
            page.goto(f"https://civitai.com/images/{iid}",
                      wait_until="networkidle", timeout=20000)
            src = page.locator("main img").first.get_attribute("src")
            results[iid] = src.split("?")[0]  # クエリ除去で原寸
        browser.close()
    return results
```

## imagehashで重複排除——300枚中41枚は同一画像の再投稿

Civitai は同一画像の再投稿が多く、初回実行では300枚中41枚(13.7%)が perceptual hash 距離4以下の重複だった。重複を学習データに混ぜると LoRA が特定構図に過学習するため、ダウンロード直後に弾く。

```python
import imagehash
from PIL import Image
from pathlib import Path

def dedup(dir: Path, threshold: int = 4) -> int:
    seen, removed = [], 0
    for f in sorted(dir.glob("*.png")):
        h = imagehash.phash(Image.open(f))
        if any(h - prev <= threshold for prev in seen):
            f.unlink(); removed += 1
        else:
            seen.append(h)
    return removed  # 実測: 300枚→41枚削除
```

## ライセンス境界と25分の実測内訳

Civitai の ToS 上、画像のスクレイピング的再配布は不可だが、自分の学習用ローカル保存は生成者が「Allow commercial use」を立てている画像に限り実務上問題になりにくい。API レスポンスの `meta` が null の画像は出所不明なので一律除外する(このルールで後述の失敗LoRA 21本のうち3本分の権利リスクを事前回避できた)。

```bash
# 実測ログ (RTX 4090機、回線1Gbps)
# API メタデータ取得 (300件+フィルタ): 4分10秒
# 画像ダウンロード 280件 (API直):      13分20秒
# Playwright フォールバック 20件:       5分40秒
# phash 重複排除:                       1分30秒
# 合計: 24分40秒 / 手動収集比 -87%
python collect.py --tag "1girl, fantasy" --want 300
```

次章では、この300枚を全部学習に使わない理由——美的スコア・顔検出・タグ一致率の3軸で上位120枚に絞る数値選定こそが Buzz 収支を分けた——を、失敗LoRA 21本の損益実数とともに開示する。
