---
title: "第2章 ZennのユーザーAPI(articles.json)をrequestsで叩きPV/スキ/コメントを構造化する"
free: false
---

## 結論: articles.jsonは1リクエストで全記事のPVを返す

Zennの`https://zenn.dev/api/articles?username=xxx`は、ログイン不要で`page_view_count`を含むJSONを返す。GraphQLでもOAuthでもなく、`requests.get`一発で全記事のPV・スキ・コメント数が取れる。まずレスポンス構造を実データで確認する。

```python
import requests

USERNAME = "your_zenn_id"
url = f"https://zenn.dev/api/articles?username={USERNAME}"
r = requests.get(url, headers={"User-Agent": "pv-tracker/1.0"}, timeout=10)
data = r.json()
print(data.keys())                 # dict_keys(['articles', 'next_page'])
print(data["articles"][0].keys())  # id, title, slug, liked_count, comments_count...
```

`articles`配列の各要素に`page_view_count`が入る。ただし非公開記事や`article_type`が`idea`の場合は欠落することがある点に注意。

## page_view_count/liked_countをdataclassで型付けする

生dictのまま回すと、フィールド名のtypoが実行時まで露見しない。`dataclass`で受け口を固定し、欠落フィールドは`dict.get`で吸収する。

```python
from dataclasses import dataclass

@dataclass
class Article:
    slug: str
    title: str
    pv: int
    liked: int
    comments: int
    article_type: str

def to_article(d: dict) -> Article:
    return Article(
        slug=d.get("slug", ""),
        title=d.get("title", "(no title)"),
        pv=d.get("page_view_count", 0),     # 欠落でも0で死なない
        liked=d.get("liked_count", 0),
        comments=d.get("comments_count", 0),
        article_type=d.get("article_type", "tech"),
    )
```

`page_view_count`が将来`view_count`へ改名されても、`KeyError`でcronが落ちず0として記録される。差分計算は次章で「前回0→今回正常値」を異常スパイクとして弾く設計にする。

## next_pageページネーションで20件超を取り切る

articles.jsonは1ページ最大しか返さず、続きは`next_page`に番号が入る。`null`になるまで`page`を増やす。

```python
def fetch_all(username: str) -> list[Article]:
    articles, page = [], 1
    while page:
        u = f"https://zenn.dev/api/articles?username={username}&page={page}"
        res = requests.get(u, headers={"User-Agent": "pv-tracker/1.0"}, timeout=10).json()
        articles += [to_article(a) for a in res["articles"]]
        page = res.get("next_page")  # Noneでループ終了
    return articles
```

記事30本でも`next_page`は2回程度しか回らず、合計レスポンスは数十KB。

## 429を避けるsleep 1.5秒とリトライ3回

連続ページ取得で稀に429が返る。180日運用で月1〜2回観測した。`time.sleep`とステータス分岐で握りつぶす。

```python
import time

def get_json(url: str, retry: int = 3) -> dict:
    for i in range(retry):
        res = requests.get(url, headers={"User-Agent": "pv-tracker/1.0"}, timeout=10)
        if res.status_code == 200:
            return res.json()
        if res.status_code == 429:
            time.sleep(1.5 * (i + 1))   # 1.5→3.0→4.5秒で後退
            continue
        res.raise_for_status()
    raise RuntimeError(f"429 over {retry} retries")
```

ページ間にも`time.sleep(1.5)`を挟めば、3ページ取得で約4.5秒。GitHub Actions無料枠の実行時間を1分も食わない。

## 取得結果を生CSV 90行に吐いて次章へ渡す

差分計算の入力は「日付+slug+PV」の3列があれば足りる。当日スナップショットをCSVに追記する。

```python
import csv, datetime

def dump_csv(articles: list[Article], path="snapshot.csv"):
    today = datetime.date.today().isoformat()
    with open(path, "a", newline="", encoding="utf-8") as f:
        w = csv.writer(f)
        for a in articles:
            w.writerow([today, a.slug, a.pv, a.liked, a.comments])

if __name__ == "__main__":
    rows = fetch_all(USERNAME)
    dump_csv(rows)
    print(f"{len(rows)} 記事 / 合計PV {sum(a.pv for a in rows)}")
```

記事30本なら毎朝30行が追記され、3日で90行。この`snapshot.csv`が、第3章でSQLiteへ取り込み「昨日比PV差分」を出す唯一の入力になる。

> 課金読者向け追記: VPS常駐ではなくGitHub Actionsのcronで回す前提なら、`snapshot.csv`はリポジトリにコミットせず`actions/cache`へ退避するのが安全だ。PVデータを公開リポジトリに平文で残すと、競合に記事ごとの伸びを読まれる。退避先の具体YAMLは第4章で扱う。
