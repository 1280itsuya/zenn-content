---
title: "第4章 SQLite＋pandasで日次KPIを自動コミット｜view/購買/CVRを差分INSERTし前日比を自動算出する集計層"
free: false
---

## 結論｜view→購買→CVRをbook_id×日付キーで冪等INSERTすれば手集計の週90分が0分になる

第3章で取得した生JSONをそのままSQLiteへ流し込むと、再実行のたびに行が二重化してCVRが半分に化ける。`UNIQUE(book_id, date)`と`UPSERT`で冪等性を担保し、深夜のCookie自己回復で同じ日付を2回叩いても壊れない集計層をこの章で確定させる。

```sql
CREATE TABLE IF NOT EXISTS kpi_daily (
  book_id   TEXT NOT NULL,
  date      TEXT NOT NULL,          -- 'YYYY-MM-DD'
  views     INTEGER NOT NULL DEFAULT 0,
  buyers    INTEGER NOT NULL DEFAULT 0,
  topic     TEXT,
  tags      TEXT,                   -- 'python,sqlite' 形式
  fetched_at TEXT NOT NULL,
  UNIQUE(book_id, date)             -- 冪等INSERTの要
);
CREATE INDEX IF NOT EXISTS idx_kpi_date ON kpi_daily(date);
```

## sqlite3のON CONFLICT UPSERTで再試行ステートマシンの二重実行を吸収する

Cookie失効リトライ後に同じ`fetch_at=03:14`の結果が再投入されても、`ON CONFLICT`で最新値に上書きするだけにする。値が増えていればviewsを更新、減っていれば（API側の一時欠損）旧値を維持する`MAX`ガードを入れる。

```python
import sqlite3

def upsert_daily(conn: sqlite3.Connection, row: dict) -> None:
    conn.execute("""
        INSERT INTO kpi_daily(book_id, date, views, buyers, topic, tags, fetched_at)
        VALUES(:book_id, :date, :views, :buyers, :topic, :tags, :fetched_at)
        ON CONFLICT(book_id, date) DO UPDATE SET
            views      = MAX(views, excluded.views),
            buyers     = MAX(buyers, excluded.buyers),
            fetched_at = excluded.fetched_at
    """, row)
    conn.commit()
```

`MAX`を挟まないと、レート制御で間引いた部分取得（views=0）が前回値を上書きし、グラフが谷になる事故を6月初週に2回起こした。

## 欠損日をdate_rangeで補間し7日移動平均の分母を割らない

Cookie失効で取得が丸ごと飛んだ日はテーブルに行が無く、移動平均の窓が4日分しか無いのに7で割って実態より低く出る。pandasの`asfreq('D')`で連続日付を生成し、欠損は前日値で前方補間する。

```python
import pandas as pd

def load_continuous(conn, book_id: str) -> pd.DataFrame:
    df = pd.read_sql(
        "SELECT date, views, buyers FROM kpi_daily WHERE book_id=? ORDER BY date",
        conn, params=(book_id,), parse_dates=["date"]
    ).set_index("date")
    df = df.asfreq("D")                 # 欠損日を行として顕在化
    df[["views", "buyers"]] = df[["views", "buyers"]].ffill().fillna(0)
    return df
```

## CVR・前日比・7日移動平均を1パスで算出する集計クエリ

view→購買のCVRと前日比（`pct_change`）、7日移動平均を1回の`assign`で出す。CVRは購買0除算を避けて`views`が0の日をNaNにし、平均からも除外する。

```python
def with_metrics(df: pd.DataFrame) -> pd.DataFrame:
    return df.assign(
        cvr        = (df.buyers / df.views.replace(0, pd.NA) * 100).round(2),
        views_dod  = df.views.pct_change().mul(100).round(1),      # 前日比%
        views_ma7  = df.views.rolling(7, min_periods=3).mean().round(1),
        buyers_ma7 = df.buyers.rolling(7, min_periods=3).mean().round(1),
    )
```

`min_periods=3`にすると、運用開始3日目から移動平均が出始め、初週から傾向を読める。CVR中央値が1.8%を割ったBookは第6章のタイトル改稿対象に回す。

## 勝ち筋分析｜伸びたtopic・tagをSQL一本でview増分順に抽出する

北極星KPIをview数に固定し、「直近7日のview増分が大きいtopic」をSQLだけで出す。tagはカンマ区切りを`json_each`で展開して集計する。

```sql
WITH delta AS (
  SELECT book_id, topic, tags,
         MAX(views) - MIN(views) AS view_gain
  FROM kpi_daily
  WHERE date >= date('now', '-7 day')
  GROUP BY book_id
)
SELECT topic,
       SUM(view_gain)              AS gain_7d,
       COUNT(*)                    AS book_cnt,
       ROUND(AVG(view_gain), 1)    AS gain_per_book
FROM delta
GROUP BY topic
ORDER BY gain_7d DESC
LIMIT 10;
```

このクエリを毎朝7時のジョブ末尾に挿し、`gain_per_book`上位3topicを次章の生成プロンプトへ自動注入する。手作業の週90分の転記とExcelピボットがこの1ファイルで消え、集計レイヤーの保守時間は0分に落ちた。
