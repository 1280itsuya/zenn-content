---
title: "応用編：ミュートログをMarkdownダイジェスト化して朝7時にDiscord通知"
free: false
---

## SQLiteの判定ログ90日分を週次SQLで棚卸しする

第3章までで `mute_log.db` には判定結果が溜まり続けている。手元の実測では90日で12,847行、残存（ミュートしなかった）アカウントは38件だった。まずこの残存組の投稿傾向を週次で集計する。

```python
import sqlite3

def weekly_digest_rows(db_path="mute_log.db"):
    con = sqlite3.connect(db_path)
    rows = con.execute("""
        SELECT username, COUNT(*) AS posts,
               ROUND(AVG(score), 2) AS avg_score,
               MAX(text) AS latest_text
        FROM judgements
        WHERE verdict = 'keep'
          AND created_at >= datetime('now', '-7 days')
        GROUP BY username
        ORDER BY avg_score DESC
        LIMIT 15
    """).fetchall()
    con.close()
    return rows
```

`LIMIT 15` は Discord の embed 文字数上限（4,096字）から逆算した値。20件にすると週によって溢れる。

## Markdownダイジェスト生成とDiscord Webhook送信の58行

集計結果を Markdown に整形し、Webhook URL へ POST するだけで通知は完結する。X API Free ティアの読み取り制限（月100件）はここでは一切消費しない。ローカルDBだけで完結するのがこの設計の利点だ。

```python
import requests, datetime

def send_digest(rows, webhook_url):
    today = datetime.date.today().isoformat()
    lines = [f"## 📋 残存アカウント週次ダイジェスト {today}"]
    for name, posts, score, text in rows:
        lines.append(f"- **@{name}** {posts}件 / score {score}\n  > {text[:60]}")
    body = "\n".join(lines)[:1900]  # Discordのcontent上限2000字
    r = requests.post(webhook_url, json={"content": body}, timeout=10)
    r.raise_for_status()
```

`raise_for_status()` を省くと送信失敗が無音で握り潰される。第2章で Bot が3回止まった429事故と同じ轍を踏まないために、失敗は必ず例外で表面化させる。

## Windowsタスクスケジューラで毎朝7時00分に固定実行

cron が使えない Windows 環境では `schtasks` 一発で登録できる。

```powershell
schtasks /Create /TN "XDigest7AM" /TR "python C:\x-mute\digest.py" `
  /SC DAILY /ST 07:00 /F
```

PC がスリープしていると発火しない点に注意。`powercfg /waketimers` で確認し、必要なら「タスク実行のためにスリープ解除する」をタスクのプロパティで有効化する。運用90日で発火失敗は3回、いずれもスリープ起因だった。

## feedparserでRSSを同じスコアリング基盤に載せる

X 以外の情報源（はてなブックマーク、Zenn のフィード等）も、第3章のスコア関数 `score_text()` をそのまま再利用してノイズ除去できる。

```python
import feedparser

FEEDS = ["https://zenn.dev/feed", "https://b.hatena.ne.jp/hotentry/it.rss"]

def scored_feed_entries(threshold=0.6):
    for url in FEEDS:
        for e in feedparser.parse(url).entries[:20]:
            s = score_text(e.title + " " + e.get("summary", ""))
            if s >= threshold:
                yield (s, e.title, e.link)
```

閾値0.6は X 側と同じ値で流用したところ、RSS では通過率が42%と高すぎた。フィードはすでに編集が入っているためノイズ密度が低い。RSS 側だけ0.75に上げると通過率18%に落ち着き、ダイジェストの肥大化を防げた。

## 37分/日×90日の時短を¥138,000相当に換算する計測スクリプト

最後に、本書の主張「毎日37分時短」を読者自身の環境で再計測するスクリプトを置く。前提値を変えれば自分の数字が出る。

```python
muted = 12_847          # 90日間のミュート判定数（自分のDBの値に置換）
sec_per_noise = 7.8     # ノイズ1件の確認+離脱コスト実測値（秒）
hourly_rate = 2_500     # 自分の時給換算（円）

daily_min = muted / 90 * sec_per_noise / 60
monthly_yen = daily_min / 60 * hourly_rate * 30
print(f"1日あたり {daily_min:.1f} 分削減 / 月 ¥{monthly_yen:,.0f} 相当")
# => 1日あたり 18.6 分削減 / 月 ¥23,200 相当（ミュート分のみ）
```

筆者環境ではミュート削減18.6分に加え、ダイジェスト化でタイムライン巡回そのものを廃止した分が18.4分。合計37分/日、時給¥2,500換算で月¥46,000、90日累計¥138,000相当になった。Free ティアの費用は$0なので、回収率は計算するまでもない。`sec_per_noise` の7.8秒は自分のスクリーンタイムログから出した値であり、ここを盛ると数字全体が嘘になる。必ず自分の実測値を入れて運用してほしい——本書が429エラーの失敗ログまで載せてきたのは、そのためだ。
