---
title: "第5章 深夜2時にcron常駐で無人運用 — Docker化・Discord通知・半年運用の数値"
free: false
---

## Xvfb + GUI Chrome を Docker で動かす理由

ヘッドレス Chrome は `navigator.webdriver` 以外に `HeadlessChrome` UA や WebGL ベンダー差異で弾かれる。第4章で抜けた CDP 接続を本番化するため、Xvfb 上で**GUI版 Chrome をそのまま起動**する。

```dockerfile
FROM python:3.12-slim
RUN apt-get update && apt-get install -y \
    xvfb google-chrome-stable fonts-noto-cjk
COPY . /app
WORKDIR /app
RUN pip install playwright==1.49.0 && playwright install-deps
# プロファイルはマウントで持ち込む(イメージに焼かない)
CMD ["xvfb-run", "-a", "-s", "-screen 0 1280x1024x24", \
     "python", "post_bot.py"]
```

`xvfb-run` で仮想ディスプレイを与えると `Browser.getVersion` の UA から `Headless` が消え、3サイトとも検知率が headless 時の約 38% → 0% に落ちた。

## 既存プロファイルを read-only マウントして cookie 失効を遅らせる

profile を毎回コンテナ内で書き換えると Chrome の整合性チェックでログアウトされる。ホスト側マスターを `:ro` で渡し、起動時に作業用へコピーする。

```bash
docker run --rm \
  -v $HOME/chrome-master:/master:ro \
  -v /tmp/chrome-work:/work \
  -e PROFILE_SRC=/master -e PROFILE_DST=/work \
  post-bot
```

この分離で cookie の平均寿命が 11 日 → 29 日に伸び、人手再ログインが月 2.6 回 → 月 1 回へ減った。

## host cron で深夜2時起動 — ロック付き起動スクリプト

二重起動すると同一プロファイルを掴んで両方クラッシュする。`flock` で排他し、深夜 02:00 に 3サイト順次投稿する。

```bash
# crontab -e
0 2 * * * /usr/bin/flock -n /tmp/postbot.lock \
  /home/itsuya/run_bot.sh >> /var/log/postbot.log 2>&1
```

```bash
#!/usr/bin/env bash
# run_bot.sh
set -euo pipefail
for SITE in pinterest blog_a blog_b; do
  docker run --rm -e SITE="$SITE" post-bot || echo "FAIL:$SITE"
  sleep $((RANDOM % 600 + 300))   # 5〜15分の人間的ゆらぎ
done
```

固定間隔は連投パターンとして弾かれるため、`RANDOM` で 300〜900 秒の揺らぎを入れるのがBAN回避の要。

## Discord Webhook に成功率を集約する

3サイト × 1日6〜7枚 = 月約 600 枚を目視確認するのは非現実的。投稿結果を Webhook に集約し、失敗のみ `@here` で鳴らす。

```python
import os, requests

def notify(site: str, ok: int, ng: int):
    rate = ok / (ok + ng) * 100 if ok + ng else 0
    mention = "@here " if ng else ""
    requests.post(os.environ["DISCORD_WEBHOOK"], json={
        "content": f"{mention}**{site}** 成功{ok}/失敗{ng} "
                   f"({rate:.0f}%)"
    }, timeout=10)
```

朝7時にスマホで成功率だけ見れば運用が回る。半年で確認時間は手作業 **月15時間 → 約20分**に圧縮された。

## 失敗キューを翌日02:05に再投稿する

`FAIL:` 行を JSON キューに退避し、翌朝の枠で先頭から消化する。CDP 切断や cookie 失効による一時失敗の約 7 割をこれで自動回収できた。

```python
import json, pathlib
Q = pathlib.Path("retry_queue.json")

def enqueue(item): 
    q = json.loads(Q.read_text() or "[]"); q.append(item)
    Q.write_text(json.dumps(q, ensure_ascii=False))

def drain(post_fn, limit=10):
    q = json.loads(Q.read_text() or "[]")
    rest = [x for x in q[:limit] if not post_fn(x)] + q[limit:]
    Q.write_text(json.dumps(rest, ensure_ascii=False))
```

## 横展開チェックリスト — 半年運用の実数

別チャンネルへ複製する前に、半年で踏んだ地雷を表で潰す。

```yaml
checklist:
  - profile_master_ro: true        # 書込でログアウト多発
  - jitter_sec: [300, 900]         # 固定間隔は3サイト中2つで凍結
  - max_per_site_day: 7            # 8枚超で blog_a が一時制限
  - cookie_refresh_interval: 29d   # 月1回の人手再ログイン前提
  - discord_alert_on: failure_only # 全通知は3日で見なくなる
metrics_6m:
  posts_total: 612
  success_rate: 94.3%   # 月1のcookie失効日に集中して低下
  manual_minutes_per_month: 20
```

成功率 94.3% の残り 5.7% はほぼ cookie 失効日の 1 日に集中する。完全無人化を狙わず「月1回・20分の再ログインだけ人が触る」設計に倒すのが、3サイトを半年止めずに回す現実解になる。
