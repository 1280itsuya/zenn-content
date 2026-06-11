---
title: "第3章: pytestで書く23の異常系テストケース設計表"
free: false
---

本のエッジ（半年で壊れた箇所と二度と壊さないテスト）を反映した第3章の本文です。

---

この章のゴールは1つ。`tests/test_probe.py` を実行すると、半年で実際に23回 probe を壊した入力が全て赤になり、`assert_sane(row)` を通った行だけが `pv.csv` に追記される状態を作る。3カテゴリ（入力異常9件・出力異常8件・冪等性6件）を `pytest.mark.parametrize` で一括検証し、KPIに0や空白が混入する経路を物理的に塞ぐ。

## 23ケースを入力異常・出力異常・冪等性で分類した設計表

第2章で記録した障害ログを起点に、再現フィクスチャと期待挙動を1行ずつ表に落とす。最多は HTTP 429（6回）、次が HTML 構造欠落（4回）だった。

```python
# tests/cases.py  カテゴリ別23ケース(発生回数つき)
CASES = [
    # category, id, fixture, expect, 第2章での発生回数
    ("input",  "http_429",     {"status": 429},                 "skip",  6),
    ("input",  "http_503",     {"status": 503},                 "skip",  3),
    ("input",  "html_missing", {"html": "<div></div>"},         "raise", 4),
    ("output", "pv_minus",     {"pv": 980, "prev": 1200},       "raise", 2),
    ("output", "likes_nan",    {"likes": "—"},                  "raise", 3),
    ("idem",   "dup_slug",     {"slug": "abc", "exists": True}, "skip",  5),
    # ... 計23行(input:9 / output:8 / idem:6)
]
```

## assert_saneで0や空をraiseして追記をスキップする

KPIを汚す犯人は「例外を握りつぶして0を書く」分岐だ。`assert_sane` は値が非数値・負・前日比マイナスなら `SaneError` を投げ、呼び出し側で追記自体を中止する。

```python
# probe/guard.py
class SaneError(ValueError): ...

def assert_sane(row: dict, prev_pv: int) -> dict:
    pv = row.get("pv")
    if not isinstance(pv, int) or pv < 0:
        raise SaneError(f"pv not int>=0: {pv!r}")
    if pv < prev_pv:                      # 前日比マイナスは計測バグ
        raise SaneError(f"pv regressed {prev_pv}->{pv}")
    if not str(row.get("likes", "")).isdigit():
        raise SaneError(f"likes not numeric: {row.get('likes')!r}")
    return row
```

## parametrizeで23ケースを1関数に畳む

23個の `def test_xxx` を書くと保守が破綻する。`ids` にケースIDを渡し、失敗時にどの障害種別が再発したか即座に分かるようにする。

```python
# tests/test_probe.py
import pytest
from probe.pipeline import handle_response
from tests.cases import CASES

@pytest.mark.parametrize(
    "cat,cid,fx,expect", [(c[0], c[1], c[2], c[3]) for c in CASES],
    ids=[f"{c[0]}:{c[1]}" for c in CASES],
)
def test_probe_does_not_corrupt_kpi(cat, cid, fx, expect, tmp_path):
    appended = handle_response(fx, csv=tmp_path / "pv.csv")
    if expect == "skip":
        assert appended is False        # 429/503/重複は追記なし
    else:                               # raise系
        assert appended is False        # 例外でも絶対に書かない
```

## CSV末尾改行欠落と日付崩れの冪等性テスト

冪等性カテゴリ6件のうち2件は「追記時の末尾改行欠落で前日行と連結」「`2026/6/9` と `2026-06-09` の混在で重複判定をすり抜け」だった。書込み前に正規化と改行保証を入れる。

```python
def append_row(path, row):
    date = row["date"].replace("/", "-")          # フォーマット統一
    line = f'{date},{row["slug"]},{row["pv"]},{row["likes"]}\n'
    with open(path, "a", encoding="utf-8", newline="") as f:
        if path.stat().st_size and not _ends_with_lf(path):
            f.write("\n")                          # 末尾改行欠落を補修
        f.write(line)

def test_no_row_concatenation(tmp_path):
    p = tmp_path / "pv.csv"
    p.write_text("2026-06-08,abc,1200,5", encoding="utf-8")  # 末尾LFなし
    append_row(p, {"date": "2026/6/9", "slug": "abc", "pv": 1300, "likes": 6})
    assert len(p.read_text().splitlines()) == 2  # 連結せず2行
```

## GitHub Actionsで23件全緑を出荷条件にする

ローカルで通っても CI で止めなければ意味がない。`pytest --maxfail=1` で23件中1件でも欠けたら失敗させ、`pv.csv` への追記コミットをブロックする。

```yaml
# .github/workflows/probe-guard.yml
- name: probe guard 23 cases
  run: |
    pytest tests/test_probe.py -q --maxfail=1 \
      --override-ini="addopts=" -p no:cacheprovider
    test $(grep -c '("' tests/cases.py) -ge 23  # ケース数の番人
```

この5本のテストが緑である限り、HTTP 429 でも likes が `—` でも `pv.csv` に0や空は1行も増えない。次章では、この緑を維持したまま probe 本体を Playwright 版へ載せ替える差分テストを設計する。
