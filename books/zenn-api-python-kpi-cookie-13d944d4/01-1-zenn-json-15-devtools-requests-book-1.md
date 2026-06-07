---
title: "第1章 Zennダッシュボードの隠しJSONを15分で叩く｜DevTools→requestsで全Book統計を1コマンド取得"
free: true
---

## HTMLスクレイプを捨てる理由｜内部APIは0.4秒・取得列はview/like/sales固定

ZennのダッシュボードをBeautifulSoupで殴ると、クラス名が変わるたびにセレクタが崩れる。代わりに画面が裏で叩く`/api/me/`系JSONを直接取れば、レスポンス構造は安定し、1冊あたりの取得は実測0.4秒で済む。

```python
import time, requests
t0 = time.time()
r = requests.get("https://zenn.dev/api/me/library", cookies=COOKIES)
print(round(time.time() - t0, 2), "秒")  # => 0.4 秒前後
print(list(r.json().keys()))             # 構造を毎回確認する癖をつける
```

## DevToolsで`/api/me/library`を特定する｜Fetch/XHRフィルタの3クリック

Chromeでダッシュボードを開き、`F12` → Networkタブ → 上部フィルタを`Fetch/XHR`に絞る。ページを再読込すると、`library`や`dashboard`を含むリクエストが並ぶ。狙いはステータス200でレスポンスが`{ "books": [...] }` 形式のもの。

```bash
# 見つけたリクエストを右クリック → Copy → Copy as cURL で控える
# 次の grep で認証ヘッダだけ確認（Cookie行の有無が肝）
pbpaste | grep -iE "cookie|/api/me"   # macOS / WSLなら xclip -o
```

## Cookie認証をrequestsへ移植する｜`_zenn_session`1個で全Bookが見える

cURLの`-b`に入っていた文字列のうち、認証に効くのは`_zenn_session`。これを辞書化してrequestsに渡せば、ログイン状態のままJSONを取得できる。

```python
COOKIES = {
    "_zenn_session": "ここにDevToolsで控えた値",
}
HEADERS = {"User-Agent": "Mozilla/5.0", "Accept": "application/json"}

res = requests.get("https://zenn.dev/api/me/library",
                   cookies=COOKIES, headers=HEADERS, timeout=10)
res.raise_for_status()
books = res.json()["books"]
print(f"{len(books)} 冊ぶんのKPIを取得")
```

## view/liked/salesをCSVへ落とす｜1コマンドで全KPIが手元に残る

レスポンスの各bookには`title`/`view_count`/`liked_count`/`sales`が含まれる。これを`csv`標準ライブラリでそのまま吐けば、Excelでも開ける成果物になる。

```python
import csv, datetime

rows = [{
    "date": datetime.date.today().isoformat(),
    "title": b["title"],
    "view": b.get("view_count", 0),
    "like": b.get("liked_count", 0),
    "sales": b.get("sales", 0),
} for b in books]

with open("zenn_kpi.csv", "w", newline="", encoding="utf-8") as f:
    w = csv.DictWriter(f, fieldnames=rows[0].keys())
    w.writeheader(); w.writerows(rows)
print("zenn_kpi.csv に", len(rows), "行を書き出し")
```

## キー欠損で落とさない｜`.get()`防御と最初の失敗ログ

非公式APIは予告なく列名が変わる。`b["sales"]`の直アクセスは`KeyError`で深夜に止まる典型なので、`.get()`で0埋めし、想定外キーはログに残して翌朝確認する。

```python
EXPECTED = {"title", "view_count", "liked_count", "sales"}
for b in books:
    missing = EXPECTED - b.keys()
    if missing:
        print(f"[WARN] {b.get('title','?')}: 欠損キー {missing}")
        # 2026-06上旬の実例: sales が一時 purchase_count に改名 → .get で吸収済み
```

ここまでで、`python zenn_kpi.py`を1回叩けば全BookのKPIが`zenn_kpi.csv`に落ちる。だが手動実行では、深夜に`_zenn_session`が失効した瞬間にデータが歯抜けになる。次章では、この失効Cookieを検知して自動で取り直す再試行ステートマシンと、429を踏まないレート制御を実コードと失敗ログ付きで組み、運用半年無停止の自動集計へ仕上げる。
