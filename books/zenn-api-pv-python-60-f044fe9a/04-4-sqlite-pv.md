---
title: "第4章 SQLiteへ日次スナップショットを蓄積しPV増分を差分検出する"
free: false
---

## article_id × fetched_date の複合主キーでスナップショットテーブルを設計する

Zenn統計APIは累計PVしか返さないため、`昨日比+47PV`を出すには取得日ごとの行を別レコードとして残すしかない。`article_id`単体をPKにすると毎日上書きされ差分が消えるので、`(article_id, fetched_date)`の複合PKにして1記事1日1行を不変保存する。

```sql
CREATE TABLE IF NOT EXISTS pv_snapshots (
  article_id   INTEGER NOT NULL,
  fetched_date TEXT    NOT NULL,   -- 'YYYY-MM-DD' (JST)
  title        TEXT,
  total_pv     INTEGER NOT NULL,
  liked_count  INTEGER NOT NULL,
  PRIMARY KEY (article_id, fetched_date)
);
CREATE INDEX IF NOT EXISTS idx_snap_date ON pv_snapshots(fetched_date);
```

## INSERT ... ON CONFLICT で日次PVをUPSERT蓄積する

cronが1日に複数回走っても重複行を作らないよう、複合PK衝突時は最新値で更新する。第2章で取得した`/api/me/articles`のJSONをそのまま流し込む。

```python
import sqlite3, datetime
def upsert(conn, articles, day=None):
    day = day or datetime.datetime.now().strftime("%Y-%m-%d")
    conn.executemany("""
      INSERT INTO pv_snapshots(article_id,fetched_date,title,total_pv,liked_count)
      VALUES(:id,:d,:title,:pv,:liked)
      ON CONFLICT(article_id,fetched_date) DO UPDATE SET
        total_pv=excluded.total_pv, liked_count=excluded.liked_count
    """, [{"id":a["id"],"d":day,"title":a["title"],
           "pv":a["page_views_count"],"liked":a["liked_count"]} for a in articles])
    conn.commit()
```

## LAG() ウィンドウ関数で前日差分と週次伸びを算出する

蓄積さえあれば増分はSQLだけで出る。`LAG(total_pv, 1)`で前回スナップショット、`LAG(total_pv, 7)`で7日前を引き、差を取る。

```sql
SELECT article_id, fetched_date, title, total_pv,
  total_pv - LAG(total_pv,1) OVER w AS daily_gain,
  total_pv - LAG(total_pv,7) OVER w AS weekly_gain
FROM pv_snapshots
WINDOW w AS (PARTITION BY article_id ORDER BY fetched_date)
ORDER BY daily_gain DESC;
```

## 累計PVが減る再集計ズレを MAX(0, …) で吸収する

Zenn側のbot除外再集計で`total_pv`が前日より減る日が月に2〜3回発生し、`daily_gain`がマイナスに振れる。負値は分析ノイズなので0でクランプし、いいね急増（前日比+5以上）の検知もこの層で行う。

```sql
SELECT title,
  MAX(0, total_pv - LAG(total_pv,1) OVER w) AS pv_gain,
  liked_count - LAG(liked_count,1) OVER w   AS like_gain
FROM pv_snapshots
WINDOW w AS (PARTITION BY article_id ORDER BY fetched_date)
HAVING like_gain >= 5            -- バズ兆候の早期検知
ORDER BY like_gain DESC;
```

## pandas で60日分から伸び率Top5を抽出する

最後に生データを資産化する。直近の`daily_gain`を伸び率（前日比%）に変換し、PVの絶対量が小さくても勢いのある記事をTop5で炙り出す。

```python
import pandas as pd, sqlite3
df = pd.read_sql("SELECT article_id,fetched_date,title,total_pv FROM pv_snapshots",
                 sqlite3.connect("zenn.db"))
df = df.sort_values(["article_id","fetched_date"])
df["gain"] = df.groupby("article_id")["total_pv"].diff().clip(lower=0)
df["prev"] = df.groupby("article_id")["total_pv"].shift(1)
df["growth_pct"] = (df["gain"] / df["prev"].replace(0, pd.NA) * 100).round(1)
latest = df[df["fetched_date"] == df["fetched_date"].max()]
print(latest.nlargest(5, "growth_pct")[["title","gain","growth_pct"]])
```

`growth_pct`でソートすると、累計300PVの新着記事が累計1万PVの古参を伸び率で上回る瞬間が見え、次に強化すべきテーマがデータで確定する。60日運用すればこのTop5が毎朝の執筆優先度リストになり、PV収集が「眺めるだけのログ」から「次の一手を決める分析基盤」に変わる。
