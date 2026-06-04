---
title: "第1章: 完成形probe.pyを動かす — 30行でZenn PV実数をSQLiteへ1コミット"
free: true
---

本章の `topics` は以下5スラッグで公開する（Zenn Book 設定用）: `claude` / `python` / `sqlite` / `automation` / `zenn`

---

`requests` と `sqlite3` だけで書いた30行の `probe.py` が、Zenn のPV実数を `kpi.db` へ毎日1行ずつINSERTする。GAも公式SDKも使わない。Cookieを1個セットして `python probe.py` を1回叩けば、あなたのBook/記事のPVがKPI DBに貯まり始める。

## probe.py 全30行をそのままコピーして動かす

以下が完成形。ファイル名は `probe.py`、Python 3.11+ で動く。

```python
import os, sqlite3, datetime, requests

COOKIE = os.environ["ZENN_COOKIE"]  # ブラウザからコピーした Cookie 文字列
URL = "https://zenn.dev/api/me/dashboard"

def fetch():
    r = requests.get(URL, headers={
        "cookie": COOKIE,
        "user-agent": "Mozilla/5.0",
    }, timeout=10)
    r.raise_for_status()  # 401/403/429 はここで例外化 → 2章で自己回復させる
    return r.json()

def save(data):
    con = sqlite3.connect("kpi.db")
    con.execute("""CREATE TABLE IF NOT EXISTS pv_daily(
        date TEXT, article_id INTEGER, slug TEXT, pv INTEGER,
        PRIMARY KEY(date, article_id))""")
    today = datetime.date.today().isoformat()
    for a in data["articles"]:
        con.execute("INSERT OR REPLACE INTO pv_daily VALUES (?,?,?,?)",
                    (today, a["id"], a["slug"], a["page_view_count"]))
    con.commit(); con.close()

if __name__ == "__main__":
    save(fetch())
    print("committed pv_daily")
```

`r.raise_for_status()` をあえて握りつぶさないのが本書の核だ。落ちることを設計に組み込む。

## ZENN_COOKIE を環境変数へ1行で渡す

Cookieは DevTools の Network タブで `/api/me/dashboard` を選び `Request Headers > cookie` を丸ごとコピーする。

```bash
# PowerShell
$env:ZENN_COOKIE = "connect.sid=s%3A...; ..."
python probe.py
# => committed pv_daily
```

## /api/me/dashboard のレスポンスJSON構造を読む

返ってくるJSONはこの形。`total_pv` と記事ごとの `page_view_count` の2粒度が同時に取れる。

```json
{
  "total_pv": 18342,
  "articles": [
    {"id": 90112, "slug": "zenn-pv-probe", "page_view_count": 421},
    {"id": 90240, "slug": "eloquent-wherehas-n1", "page_view_count": 137}
  ]
}
```

`page_view_count` がGA非依存の実数PV。これをそのまま `pv_daily.pv` に流す。

## kpi.db に入ったPVを sqlite3 で3秒で確認する

INSERT結果は標準の `sqlite3` CLI で即検証できる。

```bash
sqlite3 kpi.db "SELECT date, slug, pv FROM pv_daily ORDER BY pv DESC LIMIT 3;"
# 2026-06-04|zenn-pv-probe|421
# 2026-06-04|eloquent-wherehas-n1|137
```

`PRIMARY KEY(date, article_id)` と `INSERT OR REPLACE` により、同日に2回叩いても行は重複せず最新値で上書きされる。冪等なので Claude Code のcronから毎朝7時に呼んでも安全だ。

## 30行で得たもの、2章で外す制約

この章だけで「手元のPVが毎日勝手に貯まるDB」が完成した。だが `probe.py` には未解決の地雷が3つ残る。

```text
1. ZENN_COOKIE は数日で失効 → 401で停止
2. Zenn 側のスキーマ変更で KeyError("articles")
3. 連続実行で 429 Too Many Requests
```

この3つを「例外で落ちたら自分で直して再試行する」自己回復probeへ作り変えるのが第2章だ。Cookie自動更新・スキーマ差分検知・rate limitバックオフを実装し、cron常駐で1年放置できる計測基盤にする。30行の完成形を、落ちない運用probeへ昇格させていく。
