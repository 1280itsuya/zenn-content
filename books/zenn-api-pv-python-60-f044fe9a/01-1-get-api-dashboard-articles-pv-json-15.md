---
title: "第1章 GET /api/dashboard/articles を叩いて全記事PVをJSON取得する最小15行"
free: true
---

## Chrome DevToolsで /api/dashboard/articles を1分で特定する

結論から言う。Zennの全記事PVは公式APIドキュメントに載っていないが、ダッシュボードが内部で叩く `GET /api/dashboard/articles?page=1` を直接呼べば取れる。`https://zenn.dev/dashboard` を開き、DevTools (F12) の Network タブで `Fetch/XHR` フィルタを選び、ページを再読み込みすると一覧の先頭にこのエンドポイントが現れる。

```bash
# DevTools の Network で対象リクエストを右クリック →
# 「Copy as cURL (bash)」で得られる形。Cookie だけ確認すればよい
curl 'https://zenn.dev/api/dashboard/articles?page=1' \
  -H 'cookie: _zenn_session=XXXXXXXX...' \
  -H 'accept: application/json'
```

## page_views / liked_count を含むレスポンスJSON構造

返ってくるJSONは `articles` 配列と次ページ判定用の `next_page` を持つ。1記事あたりのキーは固定で、PV取得に必要なのは以下の3つだ。

```json
{
  "articles": [
    {
      "id": 482910,
      "title": "Zenn統計API直叩き入門",
      "slug": "zenn-dashboard-api",
      "page_views": 3284,
      "liked_count": 57,
      "published_at": "2026-05-21T07:00:12.482+09:00"
    }
  ],
  "next_page": 2
}
```

`page_views` が累計PV、`liked_count` がいいね数、`published_at` は ISO8601 (+09:00) で公開日時。`next_page` が `null` になるまで `page` をインクリメントすれば全記事を網羅できる。

## requests 15行でログイン済みCookieを載せPV配列を出力

`_zenn_session` Cookie を `requests.Session` に載せるだけで認証は通る。次の15行が全記事のタイトルとPVを標準出力に吐く最小probeだ。

```python
import os, requests

s = requests.Session()
s.headers.update({"accept": "application/json",
                  "user-agent": "Mozilla/5.0"})
s.cookies.set("_zenn_session", os.environ["ZENN_SESSION"], domain="zenn.dev")

page, rows = 1, []
while page:
    r = s.get("https://zenn.dev/api/dashboard/articles",
              params={"page": page}, timeout=10)
    r.raise_for_status()
    data = r.json()
    rows += [(a["title"], a["page_views"]) for a in data["articles"]]
    page = data["next_page"]

for title, pv in rows:
    print(f"{pv:>6}  {title}")
```

`ZENN_SESSION` 環境変数に DevTools で確認した Cookie 値を入れて実行すれば、PV降順に整える前の生データがそのまま並ぶ。

## 取得結果を pv_log.csv に書き出して成果物にする

PVをその場で見るだけでなく、CSVに落とせば表計算でもBIでも使える。`published_at` も残しておくと後章の経過日数あたりPV計算に効く。

```python
import csv, datetime

with open("pv_log.csv", "w", newline="", encoding="utf-8") as f:
    w = csv.writer(f)
    w.writerow(["fetched_at", "title", "page_views", "liked_count"])
    now = datetime.datetime.now().isoformat(timespec="seconds")
    for a in sorted(data["articles"], key=lambda x: -x["page_views"]):
        w.writerow([now, a["title"], a["page_views"], a["liked_count"]])
```

実行後 `pv_log.csv` を開けば「自分の全記事PVがCSVになった」状態になる。ここがこの本の最初の成果物だ。

## 15行probeで終わらない3つの壁 (自動Cookie/毎朝蓄積/429)

ここまでで全記事PVは取れた。ただし手動でCookieを貼る運用は長くは持たない。実際に60日回すと次の3つが必ず詰まる。

```text
壁1: _zenn_session は手動コピペだと数日で失効 → Playwright で自動ログイン取得 (第2章)
壁2: page_views は累計値のみ。日次の伸びは差分を毎朝蓄積しないと出せない (第3章)
壁3: 全記事 + 毎朝実行で page を回すと HTTP 429。間隔制御と再試行が要る (第4章)
```

この最小probeを土台に、第2章でCookie発行を自動化し、第3章で前日比PVを算出、第4章で429を回避する常駐スクリプトへ仕上げる。CSVが手に入った今、残りは「毎朝勝手に貯まる」状態へ持っていくだけだ。
