---
title: "第3章 pytest+responsesで境界値23ケースを設計しparametrizeで一括検証"
free: false
---

## 23ケースを正常3・異常9・境界7・回帰4で表に固定する

結論から言うと、テストの設計は「何を守るか」を表で先に固定し、その表をそのまま `parametrize` の引数へ落とす。計測probeが静かに壊れる事故は、入力の境界(views=0/None、HTML構造変化)を網羅していないことが原因なので、内訳を数で約束する。

| 分類 | 件数 | 代表ケース |
|---|---|---|
| 正常系 | 3 | views=1234, slug一致, 追記成功 |
| 異常系 | 9 | 404, タイムアウト, 空HTML, 認証切れ |
| 境界値 | 7 | views=0, None, 最大桁, 全角数字 |
| 回帰固定 | 4 | 過去の事故HTMLをゴールデン化 |

```python
# cases.py — 23ケースを単一のソースに集約
CASES = [
    ("ok_normal", "<span>1,234</span>", 1234),
    ("zero", "<span>0</span>", 0),       # 0 と None は別物
    ("missing", "<div></div>", None),    # 要素欠落
]
```

## @pytest.mark.parametrizeで23ケースを1関数に圧縮する

23個のテスト関数を手書きせず、`ids` を付けて失敗時にケース名が出るようにする。

```python
import pytest
from probe import parse_views
from cases import CASES

@pytest.mark.parametrize(
    "name,html,expected", CASES, ids=[c[0] for c in CASES]
)
def test_parse_views(name, html, expected):
    assert parse_views(html) == expected
```

`pytest -q` の出力に `test_parse_views[zero]` と並び、どの境界が落ちたか即座に特定できる。

## responsesでZennのHTTPを偽装しviews=0とNoneを区別する

知らないと損するのが、`views=0` を `None`(取得失敗)に丸めるバグ。これがKPIを偽の横ばいにする。`responses` で実ネットワークを遮断して両者を分岐させる。

```python
import responses, pytest

@responses.activate
@pytest.mark.parametrize("body,expected", [
    ('{"page_views":0}', 0),
    ('{}', None),  # キー欠落は None、0 ではない
])
def test_fetch(body, expected):
    responses.add(responses.GET,
        "https://zenn.dev/api/probe", body=body, status=200)
    assert fetch_views("probe") == expected
```

`0 is not None` を assert で踏み抜けるので、丸めバグはCIで赤になる。

## HTMLゴールデンファイルでZennの構造変化を検知する

Zenn側のDOMが変わるとprobeは例外も出さず横ばいを返す。過去に壊れた実HTMLを `golden/` に保存し、回帰固定4ケースで突き合わせる。

```python
from pathlib import Path

@pytest.mark.parametrize("fixture", Path("golden").glob("*.html"))
def test_golden(fixture):
    expected = int(fixture.with_suffix(".expected").read_text())
    assert parse_views(fixture.read_text()) == expected
```

構造変化でパースが崩れた瞬間に該当ファイル名で落ちるため、原因HTMLが手元に残る。

## tmp_pathで同日同slug上書きの冪等性を検証する

計測ログCSVは「同日・同slugは上書き、別日は追記」が崩れると二重計上で数字が膨らむ。`tmp_path` で毎回新規CSVを作り副作用を断つ。

```python
def test_idempotent(tmp_path):
    csv = tmp_path / "views.csv"
    record(csv, "2026-06-03", "probe", 100)
    record(csv, "2026-06-03", "probe", 120)  # 同日同slug
    rows = csv.read_text().strip().splitlines()
    assert len(rows) == 1          # 追記されない
    assert rows[0].endswith(",120")  # 最新で上書き
```

## pytest-covで未到達分岐を可視化する

最後に、23ケースで「どの分岐が一度も実行されていないか」を `--cov-branch` で炙り出す。

```bash
pytest --cov=probe --cov-branch --cov-report=term-missing
# probe.py  92%  Missing: 47->49 (views=None の早期return)
```

`Missing` に出た行番号が、まだ守れていない事故経路そのもの。23ケースに不足分を追記してこの行を消し込めば、KPIの偽横ばいは恒久的に塞がる。
