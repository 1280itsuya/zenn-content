---
title: "第3章 Pinterest/Adobe Stock/Boothのフォーム差分を吸収する共通Poster設計"
free: false
---

## Pinterest/Adobe Stock/Boothの3フォームをDOM単位で比較した差分表

3サイトのアップロードUIは「ファイル投入方法」「必須メタ」「セレクタの安定性」が全部違う。第2章でCDP接続した既存Chromeプロファイルを使い回す前提で、まず差分を確定させる。

| 項目 | Pinterest | Adobe Stock | Booth |
|---|---|---|---|
| ファイル投入 | `input[type=file]`(drag&drop偽装) | `input[type=file]`複数 | 商品画像+本体ファイル別枠 |
| 必須メタ | title/board | category/keyword 5語以上 | 商品名/価格/種別 |
| セレクタ | `data-test-id`安定 | 動的`id="comp-xxxx"` | `name`属性安定 |
| 完了判定 | `/v3/pins/` 200 | `/uploads` 200 | redirect to `/items/{id}` |

この表のうち最も事故るのが Adobe Stock の動的id。次節の共通設計で `data-testid`/XPath代替に逃がす。

## 抽象基底class Posterで upload/fill_meta/submit を3メソッドに固定する

差分を吸収する核は、サイト共通の3フェーズを基底classで強制すること。サブクラスは差分のあるメソッドだけ上書きする。

```python
from abc import ABC, abstractmethod
from playwright.sync_api import Page

class Poster(ABC):
    site: str
    upload_url: str

    def post(self, page: Page, asset: dict) -> str:
        page.goto(self.upload_url, wait_until="domcontentloaded")
        self.upload(page, asset["file"])
        self.fill_meta(page, asset)
        return self.submit(page)  # 投稿IDを返す

    @abstractmethod
    def upload(self, page: Page, file: str): ...
    @abstractmethod
    def fill_meta(self, page: Page, asset: dict): ...
    @abstractmethod
    def submit(self, page: Page) -> str: ...
```

`post()` だけがオーケストレータから呼ばれる。3サイト分の差分はサブクラス3つに閉じ込まる。

## Pinterestのdrag&dropを set_input_files で迂回し64枚を流す

Pinterestはdrag&drop UIだが、裏に隠れた `input[type=file]` へ直接 `set_input_files` すればドラッグ座標計算は不要。深夜バッチで64枚を回す際の安定度が段違いになる。

```python
class PinterestPoster(Poster):
    site = "pinterest"
    upload_url = "https://www.pinterest.com/pin-builder/"

    def upload(self, page, file):
        page.set_input_files("input[type=file]", file)
        page.wait_for_selector("div[data-test-id='media-preview'] img",
                               timeout=20000)

    def fill_meta(self, page, asset):
        page.fill("textarea#pin-draft-title", asset["title"])
        page.click(f"div[aria-label='{asset['board']}']")

    def submit(self, page) -> str:
        with page.expect_response(lambda r: "/v3/pins/" in r.url
                                  and r.request.method == "POST") as info:
            page.click("button[data-test-id='board-dropdown-save-button']")
        return info.value.json()["resource_response"]["data"]["id"]
```

## Adobe Stockの動的id `comp-xxxx` を data-testid とXPathで吸収する

Adobe Stockは `id="comp-8f3a"` のようにビルド毎に変わるidを吐く。CSSの`id`セレクタは即死するので、`data-testid` か、ラベルテキスト起点のXPathに切り替える。キーワードは5語未満だと審査前に弾かれるため、補填ロジックを入れる。

```python
class AdobeStockPoster(Poster):
    site = "adobe_stock"
    upload_url = "https://contributor.stock.adobe.com/uploads"

    def fill_meta(self, page, asset):
        kws = asset["keywords"]
        if len(kws) < 5:                      # 5語未満は審査落ち
            kws = (kws + ["AI", "art", "digital", "design", "render"])[:5]
        # 動的idを避けラベル文言からXPathで掘る
        page.locator("xpath=//label[contains(.,'Category')]/following::select[1]"
                     ).select_option(label=asset["category"])
        page.fill("input[data-testid='keyword-input']", ", ".join(kws))
```

## 完了判定をネットワーク応答で取りリトライ＋失敗スクショで深夜無人化する

`time.sleep` で待つと深夜の負荷変動でBAN誘発級の連打になる。`expect_response` で実応答を待ち、失敗時は3回リトライ＋スクショ保存。これで朝にログだけ見れば原因が確定する。

```python
import time, pathlib

def safe_post(poster: Poster, page, asset, retry=3) -> str | None:
    for i in range(retry):
        try:
            return poster.post(page, asset)
        except Exception as e:
            ts = int(asset.get("seq", i))     # Date.now不可な環境用に連番
            shot = pathlib.Path(f"logs/{poster.site}_{ts}_{i}.png")
            page.screenshot(path=str(shot), full_page=True)
            print(f"[{poster.site}] retry {i+1}/{retry} {type(e).__name__}: {e}")
            time.sleep(8 * (i + 1))           # 8/16/24秒バックオフ
    return None
```

3サイトとも `safe_post` 経由に統一すれば、Pinterestの`/v3/pins/`応答もAdobe Stockの審査キュー投入も同じ失敗ハンドリングに乗る。月600枚（3サイト×約200枚）を無人で回す土台が、この基底class＋応答待ち＋スクショの3点で完成する。
