---
title: "第5章 click→売上をつなぐ:日次集計バッチとnote/Zenn記事別ROI可視化"
free: false
---

## clickを売上に変換する daily_rollup バッチを cron で回す

EPCを出すには、まず生ログの`/go?id=`クリックを「記事id×日付」へ畳む必要がある。`clicks`テーブル(第3章で作成)を日次で`daily_clicks`へ集計するバッチを書く。実行は毎朝6時、cronで`daily_rollup.py`を叩く。

```python
# daily_rollup.py
import sqlite3, datetime
db = sqlite3.connect("kpi.db")
y = (datetime.date.today() - datetime.timedelta(days=1)).isoformat()
db.execute("""
INSERT INTO daily_clicks(article_id, dt, click)
SELECT article_id, date(ts) dt, COUNT(*) FROM clicks
WHERE date(ts)=? AND is_bot=0
GROUP BY article_id, dt
ON CONFLICT(article_id, dt) DO UPDATE SET click=excluded.click
""", (y,))
db.commit()
```

`ON CONFLICT`で再実行しても二重加算しない。`is_bot=0`で第4章のBot除外を引き継ぐ。

```bash
# crontab -e
0 6 * * * cd /opt/kpi && /opt/kpi/.venv/bin/python daily_rollup.py >> rollup.log 2>&1
```

## ASP確定報酬CSVをimportしてEPCとCVRを記事別に算出する

A8.netの確定報酬CSVを`revenues`へ取り込み、clickと突き合わせる。EPC(1クリックあたり収益)とCVRを記事別に出すSQLがこの章の核心だ。

```sql
SELECT c.article_id,
       SUM(c.click)                         AS clicks,
       SUM(r.amount)                         AS revenue,
       ROUND(SUM(r.amount)*1.0/SUM(c.click),1) AS epc,
       ROUND(COUNT(r.id)*100.0/SUM(c.click),2) AS cvr_pct
FROM daily_clicks c
LEFT JOIN revenues r ON r.article_id=c.article_id
WHERE c.dt >= date('now','-30 day')
GROUP BY c.article_id
ORDER BY epc DESC;
```

30日窓で集計することで、単発スパイクではなく定常EPCで記事を順位付けできる。CVRはコンバージョン件数÷clickなので、clickが10未満の記事はノイズとして後段で除外する。

## EPC¥0記事を機械検出しリンク差し替え候補をMarkdownで自動生成する

30日でclickが20以上あるのにrevenueが¥0の記事は、リンク選定ミスの確度が高い。これを自動抽出し、差し替え候補レポートを吐く。

```python
rows = db.execute("""
SELECT article_id, SUM(click) clk FROM daily_clicks
WHERE dt>=date('now','-30 day') GROUP BY article_id
HAVING clk>=20 AND article_id NOT IN
  (SELECT article_id FROM revenues WHERE amount>0)
""").fetchall()
with open("report.md","w",encoding="utf-8") as f:
    f.write("# EPC¥0 差し替え候補\n\n")
    for aid, clk in rows:
        f.write(f"- `{aid}` clicks={clk} → 高EPC案件へ差し替え\n")
```

clickは付いているのに収益が出ない=需要はあるがオファーが弱い記事。ここを直すのが最短の収益改善になる。

## EPC上位2記事へ配分転換し収益を1.6倍にした3か月の実数値

3か月運用した結果を実数で公開する。全58記事のうちEPC上位2記事(EPC¥41.2と¥38.7)へ内部リンクとCTA位置を寄せ、EPC¥0の14記事のリンクを差し替えた。

| 月 | 総click | revenue | 平均EPC |
|----|--------|---------|--------|
| 1月目 | 3,120 | ¥9,180 | ¥2.9 |
| 3月目 | 3,050 | ¥14,700 | ¥4.8 |

```sql
-- 配分転換の効果を月次で検証
SELECT strftime('%Y-%m', dt) m, SUM(click) clk,
       (SELECT SUM(amount) FROM revenues r
        WHERE strftime('%Y-%m',r.dt)=strftime('%Y-%m',c.dt)) rev
FROM daily_clicks c GROUP BY m;
```

総clickは3,120→3,050とほぼ横ばい。にもかかわらずrevenueは¥9,180→¥14,700で1.6倍になった。集客を増やさず配分を変えただけで収益が伸びる——これが自前click計測を意思決定へ接続した実利益だ。

## 記事別ROIダッシュボードをdatasette 1コマンドで公開する

最終形は、毎朝の`daily_rollup.py`が更新する`kpi.db`をそのまま`datasette`でブラウザ表示する構成だ。BIツールもGA連携も不要で、SQLビューがそのままダッシュボードになる。

```bash
pip install datasette
datasette kpi.db --setting sql_time_limit_ms 3000 -p 8765
```

EPC降順ビューを保存しておけば、毎朝6時のバッチ後に「今日伸ばすべき2記事」と「差し替えるべき記事」が一覧で確定する。GAの管理画面を開かず、302リダイレクト1ホップで自前加算したclick列だけで、記事単位のROI判断が完結する計測基盤になる。
