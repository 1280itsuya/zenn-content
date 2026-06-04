---
title: "第3章: スキーマ崩れで落とす例外設計 — Pydanticで非公開APIの変更を即検知"
free: false
---

## dict.get() を捨て Pydantic の BaseModel で型を固定する

非公開 Zenn 統計 API のレスポンスは `dict.get("articles", [])` で受けてはいけない。キーが消えても空配列が返り、PV がゼロに化けたまま KPI DB に記録され続けるからだ。`BaseModel` で型と必須キーを宣言し、ズレた瞬間に `ValidationError` で probe を止める。

```python
from pydantic import BaseModel, ConfigDict

class ArticleStat(BaseModel):
    model_config = ConfigDict(extra="forbid")  # 未知キーで即失敗
    id: int
    title: str
    page_views_count: int  # この型不一致も検知対象
```

## articles → items 改名を ValidationError で検知した2026年某月の事例

`extra="forbid"` を入れていたおかげで、レスポンスの `articles` 配列が `items` へ改名された日に probe が落ちた。ログには欠測ではなく `ValidationError` が残り、5分で原因を特定できた。

```python
class StatsResponse(BaseModel):
    model_config = ConfigDict(extra="forbid")
    items: list[ArticleStat]  # 旧: articles。改名時は次行が ValidationError

# 1 validation error for StatsResponse
# articles: Extra inputs are not permitted [type=extra_forbidden]
```

## Optional 濫用が欠測を握りつぶす罠

`page_views_count: int | None = None` のように Optional を多用すると、API がキーを落としても `None` が通ってしまい、改名・廃止に気づけない。本当に欠けうるフィールドだけ Optional にし、PV のようなコア指標は非 Optional の `int` で固定する。

```python
class ArticleStat(BaseModel):
    model_config = ConfigDict(extra="forbid")
    id: int
    page_views_count: int       # 欠けたら落とす(Optional禁止)
    comments_count: int = 0     # 仕様上ゼロ相当はデフォルトで許容
```

## 欠測日を NULL で埋めず行ごと落とす SQLite KPI DB

`ValidationError` の日は行を作らない。NULL で埋めるとグラフが0ではなく「線が途切れる」ため、欠測に目で気づける。`UNIQUE` 制約で再実行時の二重記録も防ぐ。

```sql
CREATE TABLE IF NOT EXISTS kpi_pv (
  measured_on TEXT NOT NULL,
  article_id  INTEGER NOT NULL,
  page_views  INTEGER NOT NULL,        -- NOT NULL: 欠測日は行ごと欠落
  UNIQUE(measured_on, article_id)      -- 同日同記事の重複INSERTを拒否
);
```

```python
def save(conn, day: str, stats: StatsResponse) -> None:
    conn.executemany(
        "INSERT OR IGNORE INTO kpi_pv VALUES (?,?,?)",
        [(day, a.id, a.page_views_count) for a in stats.items],
    )
    conn.commit()
```

## ValidationError で probe 全体を落として自己回復させる

握りつぶさず例外を上げ、cron のリトライと通知に回す。これが「あえて落として自己回復させる」運用の核だ。

```python
import sys
from pydantic import ValidationError

try:
    stats = StatsResponse.model_validate(raw_json)
    save(conn, day, stats)
except ValidationError as e:
    notify_slack(f"Zenn API schema drift:\n{e}")  # 欠測ではなく障害として扱う
    sys.exit(1)  # cron が翌実行で再試行、KPIは欠落行で可視化
```

`sys.exit(1)` により次回 cron までは欠測行が残り、グラフの断線がそのまま「スキーマ変更日」のマーカーになる。dict ベースの probe では永久に得られない、自己診断する KPI パイプラインが完成する。

---

topics: `["zenn", "python", "pydantic", "sqlite", "automation"]`

※ 指定スラッグのうち `claude` を含めた5個構成にする場合は `["claude", "python", "sqlite", "automation", "zenn"]` を採用（いずれも Zenn 有効スラッグ）。
