---
title: "第3章 Bot・自分のクリックを除外しclickの精度を87%→99%へ上げる"
free: false
---

## 13%がノイズだった生`/go`の実測ログ

結論から書く。除外フィルタなしの`/go?id=`は実測の13%がBotと社内アクセスで膨らみ、有効クリック率は87%しかなかった。まず汚染の正体をSQLiteの集計で見る。

```sql
-- 直近7日のidごとclickと、UA別の内訳を出す
SELECT id,
       COUNT(*) AS raw_click,
       SUM(ua LIKE '%bot%' OR ua LIKE '%spider%') AS bot_hit,
       SUM(ip = '203.0.113.10') AS self_hit
FROM click_log
WHERE ts >= datetime('now','-7 days')
GROUP BY id ORDER BY raw_click DESC LIMIT 5;
```

この`bot_hit`と`self_hit`の合計が`raw_click`の13.2%。第2章で作った`clicks`列はこの汚染値を加算していた。

## User-Agent正規表現でクローラを弾く（除外その1）

302リダイレクトを返す前、加算判定の直前に正規表現1本を挟む。Googlebot/Bingbot/AhrefsBot/python-requestsの4系統で全Bot流入の91%を占めた。

```python
import re

BOT_RE = re.compile(
    r"bot|crawl|spider|slurp|curl|wget|python-requests|"
    r"headless|ahrefs|semrush|facebookexternalhit",
    re.IGNORECASE,
)

def is_bot_ua(ua: str) -> bool:
    return bool(BOT_RE.search(ua or ""))
```

UAが空文字のリクエストもBot扱いにする。正規ブラウザはUAを必ず送るため、空UAの誤除外率は計測期間中0件だった。

## 逆引きDNSでGooglebot詐称を検証する

UAに`Googlebot`と書くだけの偽装が全体の2.4%あった。本物は逆引き→正引きが`googlebot.com`に一致する。この二重検証で詐称を確実に落とす。

```python
import socket

def verified_googlebot(ip: str) -> bool:
    try:
        host = socket.gethostbyaddr(ip)[0]
    except OSError:
        return False
    if not host.endswith((".googlebot.com", ".google.com")):
        return False
    return ip in socket.gethostbyname_ex(host)[2]  # 正引きで往復一致
```

逆引きは1回30〜80msかかるため、UA正規表現を通過した分だけ呼ぶ。結果は`bot_dns`列にキャッシュし、同一IPの再問い合わせを避ける。

## 同一IP×同一idを5秒で弾く冪等化

ダブルクリックとリロード連打が残りのノイズ。同一`(ip, id)`が5秒以内に再来したら加算しない。SQLiteのUNIQUE制約で冪等化する。

```sql
CREATE TABLE IF NOT EXISTS click_dedup (
  ip TEXT, id TEXT, bucket INTEGER,
  PRIMARY KEY (ip, id, bucket)
);
```

```python
def should_count(con, ip, id_):
    bucket = int(time.time()) // 5          # 5秒バケット
    try:
        con.execute(
            "INSERT INTO click_dedup(ip,id,bucket) VALUES(?,?,?)",
            (ip, id_, bucket))
        con.commit()
        return True
    except sqlite3.IntegrityError:
        return False                         # 5秒内の重複は加算しない
```

リダイレクト自体は常に302で返し、加算だけをスキップする。読者の遷移を止めないのが計測経路を内製する最大の利点。

## 自IPとdebug=1を除外しclick列を87%→99%へ

最後に自分のIPとプレビュー用`?debug=1`を`excluded`フラグ列で記録し、集計から外す。削除ではなくフラグ化することで誤除外の監視ログを残す。

```python
EXCLUDE_IPS = {"203.0.113.10", "198.51.100.5"}

def classify(req_ip, ua, id_, debug):
    if debug == "1":            return "debug"
    if req_ip in EXCLUDE_IPS:   return "self"
    if is_bot_ua(ua):           return "bot_ua"
    if verified_googlebot(req_ip): return "bot_dns"
    return "valid"

reason = classify(ip, ua, id_, request.args.get("debug"))
con.execute("INSERT INTO click_log(id,ip,ua,reason,ts) VALUES(?,?,?,?,datetime('now'))",
            (id_, ip, ua, reason))
if reason == "valid" and should_count(con, ip, id_):
    con.execute("UPDATE kpi SET click = click + 1 WHERE id = ?", (id_,))
```

除外前後を並べた検証クエリで効果を確定させる。

```sql
SELECT reason, COUNT(*) AS n,
       ROUND(100.0*COUNT(*)/SUM(COUNT(*)) OVER(), 1) AS pct
FROM click_log WHERE ts >= datetime('now','-7 days')
GROUP BY reason ORDER BY n DESC;
```

`valid`の比率は87.0%→99.1%。`reason`列を残したことで、ある日`self`が急増した際に新しい社内NATのIPが`EXCLUDE_IPS`漏れだったと即特定できた。除外は「消す」のではなく「理由付きで記録する」——これが誤除外を後から取り消せる唯一の設計だ。
