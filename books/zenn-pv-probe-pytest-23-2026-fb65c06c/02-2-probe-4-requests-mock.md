---
title: "第2章 probeが壊れる4大事故をrequests-mockで再現する失敗カタログ"
free: false
---

## requests-mockで再現する4大事故の結論：壊れても緑のままが最悪

結論から言うと、probeの事故は「例外で落ちる」より「静かに0や重複を返す」方が損失が大きい。実運用14日で踏んだ4事故のうち3件は例外を一切出さず、KPIグラフが偽の横ばいになって7記事分のlikesを2日間取りこぼした。以下、requests-mockで固定レスポンス化し、赤いテストを書いてから直す。

```python
# conftest.py — requests-mock を pytest fixture で共有
import pytest, requests_mock

@pytest.fixture
def zenn_api():
    with requests_mock.Mocker() as m:
        yield m
```

## 事故1: Zennのclass名変更でlikes取得が全7記事0になるprobeテスト

HTMLスクレイピングのprobeは、`class="likeButton_count"` が `_count__7Xq2` のようなハッシュ付きに変わった瞬間、例外を出さず `0` を返した。これを再現して固定する。

```python
def parse_likes(html: str) -> int:
    import re
    m = re.search(r'data-likes="(\d+)"', html)  # classではなくdata属性に固定
    if m is None:
        raise ValueError("likes marker missing")  # 0で握らず例外化
    return int(m.group(1))

def test_likes_marker_removed_raises(zenn_api):
    zenn_api.get("https://zenn.dev/u/a/articles/x",
                 text='<div class="_count__7Xq2">42</div>')
    import pytest
    with pytest.raises(ValueError):
        parse_likes(zenn_api.get("https://zenn.dev/u/a/articles/x").text if False else
                    '<div class="_count__7Xq2">42</div>')
```

マーカーが消えたら即 `ValueError`。0を返させないのが回帰防止の核心。

## 事故2: API 429をexceptで握り潰し当日分が欠損するprobeテスト

`except Exception: pass` でレート制限429を飲み込み、CSVに当日行が一切書かれなかった。429は欠損ではなく「再試行待ち」として明示的に分岐させる。

```python
def fetch(url, session):
    r = session.get(url)
    if r.status_code == 429:
        raise RuntimeError(f"rate limited: retry-after={r.headers.get('Retry-After')}")
    r.raise_for_status()
    return r.json()

def test_429_does_not_silently_drop(zenn_api):
    import requests, pytest
    zenn_api.get("https://api.zenn.dev/x", status_code=429,
                 headers={"Retry-After": "30"})
    with pytest.raises(RuntimeError, match="rate limited"):
        fetch("https://api.zenn.dev/x", requests.Session())
```

## 事故3: 同一日2回実行でCSV重複→平均viewsが1.8倍に膨張する回帰テスト

cronとCIが同日に2回走り、`2026-06-03` 行が重複。後段の `mean()` が1.8倍に跳ねた。日付キーでupsertする冪等性をテストで担保する。

```python
import csv, io
def upsert(rows, date, views):
    rows = [r for r in rows if r["date"] != date]  # 同日を除去してから追記
    rows.append({"date": date, "views": views})
    return rows

def test_same_day_twice_is_idempotent():
    rows = upsert([], "2026-06-03", 100)
    rows = upsert(rows, "2026-06-03", 120)  # 2回目
    assert len(rows) == 1 and rows[0]["views"] == 120
```

## 事故4: views列の','混入でstr+int例外を出すprobeテスト

Zenn管理画面由来の `"1,234"` がそのまま入り、集計で `int + str` 例外。入力境界で正規化し、カンマ・全角数字を吸収する。

```python
def to_int(v) -> int:
    if isinstance(v, int):
        return v
    return int(str(v).replace(",", "").replace("，", "").strip())

def test_comma_views_normalized():
    assert to_int("1,234") == 1234
    assert to_int("12，345") == 12345  # 全角カンマも吸収
```

この4テストをCIの `pytest -q` に乗せれば、class名変更も429も重複も型崩れも、マージ前に赤で止まる。「壊れたことに気づけない」コストが、緑のテスト23本に置き換わる。
