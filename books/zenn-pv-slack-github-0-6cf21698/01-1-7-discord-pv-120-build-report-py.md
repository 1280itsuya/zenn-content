---
title: "第1章 完成形プレビュー：毎朝7時にDiscordへ届くPV差分通知と120行のbuild_report.py"
free: true
---

## 結論：Zennが出さない「日次PV差分」を120行で毎朝Discordに流す

Zenn公式アナリティクスは累計PVのグラフは出すが、`nuxt記事 +143PV(前日比+38%)`のような記事別の日次差分も週間ランキングも出さない。本書はこの欠落を、非公式statsエンドポイント＋SQLiteスナップショット差分で埋める。必要なのはGitHubアカウントとDiscord Webhook 1本だけ、追加課金は0円だ。下が実際に毎朝7時へ届く通知の中身である。

```text
📊 Zenn PV差分 2026-06-05 07:00
1位 nuxt記事        1432PV  (+143 / +38%)
2位 sqlite-snapshot  988PV  (+51  / +5.4%)
3位 cron-drift対策   774PV  (+12  / +1.5%)
昨日の総PV +206 / 観測記事 18本
```

## build_report.py 全120行：取得→差分→Discordまで1ファイル

下のスクリプトをコピペし、`ZENN_USER`と`DISCORD_WEBHOOK`を埋めれば動く。`articles.db`に前回値を保存し、今回との差分だけを送る構造だ。

```python
import os, json, sqlite3, datetime, urllib.request

ZENN_USER = os.environ["ZENN_USER"]
WEBHOOK   = os.environ["DISCORD_WEBHOOK"]
DB        = "articles.db"
API       = f"https://zenn.dev/api/articles?username={ZENN_USER}&order=latest"

def fetch_articles():
    req = urllib.request.Request(API, headers={"User-Agent": "pv-report/1.0"})
    with urllib.request.urlopen(req, timeout=20) as r:
        data = json.load(r)
    # 非公式statsエンドポイント。page_views は公開記事のみ返る
    return [
        {"slug": a["slug"], "title": a["title"], "pv": a.get("page_views", 0)}
        for a in data["articles"]
    ]

def init_db():
    con = sqlite3.connect(DB)
    con.execute("""CREATE TABLE IF NOT EXISTS snap(
        slug TEXT, day TEXT, pv INTEGER, title TEXT,
        PRIMARY KEY(slug, day))""")
    con.commit()
    return con

def yesterday_pv(con, slug, today):
    # 直近で today より前の最新スナップショットを1件
    row = con.execute(
        "SELECT pv FROM snap WHERE slug=? AND day<? ORDER BY day DESC LIMIT 1",
        (slug, today)).fetchone()
    return row[0] if row else None

def save_snapshot(con, arts, today):
    for a in arts:
        con.execute(
            "INSERT OR REPLACE INTO snap(slug,day,pv,title) VALUES(?,?,?,?)",
            (a["slug"], today, a["pv"], a["title"]))
    con.commit()

def build_diff(con, arts, today):
    rows = []
    for a in arts:
        prev = yesterday_pv(con, a["slug"], today)
        delta = None if prev is None else a["pv"] - prev
        rate  = None if not prev else round(delta / prev * 100, 1)
        rows.append({**a, "delta": delta, "rate": rate})
    # 差分が出た記事だけを増加順に
    diffed = [r for r in rows if r["delta"] not in (None, 0)]
    return sorted(diffed, key=lambda r: r["delta"], reverse=True)

def format_message(rows, today):
    total = sum(r["delta"] or 0 for r in rows)
    lines = [f"📊 Zenn PV差分 {today} 07:00"]
    for i, r in enumerate(rows[:3], 1):
        rate = f"{r['rate']:+}%" if r["rate"] is not None else "n/a"
        lines.append(f"{i}位 {r['title'][:14]:　<14} "
                     f"{r['pv']}PV ({r['delta']:+} / {rate})")
    lines.append(f"昨日の総PV {total:+} / 観測記事 {len(rows)}本")
    return "\n".join(lines)

def already_sent(con, today):
    con.execute("""CREATE TABLE IF NOT EXISTS sent(day TEXT PRIMARY KEY)""")
    if con.execute("SELECT 1 FROM sent WHERE day=?", (today,)).fetchone():
        return True
    con.execute("INSERT INTO sent(day) VALUES(?)", (today,))
    con.commit()
    return False

def notify(text):
    body = json.dumps({"content": text}).encode()
    req = urllib.request.Request(
        WEBHOOK, data=body, headers={"Content-Type": "application/json"})
    urllib.request.urlopen(req, timeout=20)

def main():
    today = datetime.date.today().isoformat()
    con = init_db()
    if already_sent(con, today):     # cron二重発火による通知重複を握り潰す
        print("skip: already sent", today)
        return
    arts = fetch_articles()
    rows = build_diff(con, arts, today)
    save_snapshot(con, arts, today)
    if not rows:
        print("skip: no diff")
        return
    notify(format_message(rows, today))
    print("sent:", len(rows), "articles")

if __name__ == "__main__":
    main()
```

## already_sent()が握り潰す「cron二重発火」180日目の実バグ

GitHub Actionsの`schedule`はUTC基準で、`cron: '0 22 * * *'`を日本時間7時のつもりで置くと、ランナー混雑時に2〜4分のdriftが乗り、稀に同一ジョブが二重起動して通知が2通届く。180日運用で14回踏んだ。対策は上の`already_sent()`で、`sent`テーブルに当日キーを書き込み済みなら送信をスキップする。日付キーで冪等にするのが効く。

```yaml
# .github/workflows/pv-report.yml
on:
  schedule:
    - cron: '0 22 * * *'   # UTC22:00 = JST07:00。driftは ±4分まで観測
permissions:
  contents: write          # articles.db を commit して翌朝の差分基準にする
```

## SQLiteスナップショットを差分基準にする理由（CSVだと壊れた）

当初は`history.csv`に追記していたが、ランナーが途中で落ちると行が欠けて差分計算が`KeyError`で死んだ。`PRIMARY KEY(slug, day)`を持つSQLiteへ`INSERT OR REPLACE`する方式に変えてから、再実行しても同じ日付は上書きされ、欠損行は出なくなった。`yesterday_pv()`が「today より前の最新1件」を引くので、土日にActionsが間引かれて1日抜けても、前々日との差分へ自動で繋がる。

```bash
# ローカル即時テスト：本番Webhookを汚さず差分ロジックだけ確認
ZENN_USER=your_id \
DISCORD_WEBHOOK=https://discordapp.com/api/webhooks/0/test \
python build_report.py
sqlite3 articles.db "SELECT day, slug, pv FROM snap ORDER BY day DESC LIMIT 5;"
```

## 第2章以降で埋める3つの穴：差分取得・cron・通知整形

この120行は動くが、3点が未完成だ。第2章は`page_views`が非公開記事や直近公開記事で欠落するときの補完ロジック、第3章はUTC/JST変換とActions無料枠（月2,000分）を使い切らない実行設計、第4章はDiscordの2,000文字制限と全角パディング崩れを直す整形を扱う。下のチェックで自分の環境の前提だけ先に確認しておくとよい。

```bash
# 前提チェック：Webhook疎通とZenn API応答を5秒で確認
curl -s -o /dev/null -w "webhook:%{http_code}\n" \
  -X POST -H "Content-Type: application/json" \
  -d '{"content":"setup ok"}' "$DISCORD_WEBHOOK"
curl -s "https://zenn.dev/api/articles?username=$ZENN_USER" | head -c 200
```

ここまでで「完成形」と「コピペで動く本体」は手元に揃った。残る3つの穴を埋めれば、明日の朝7時には自分のアカウントのPV差分がDiscordに並ぶ。
