---
title: "第5章 cron+GitHub Actionsで毎朝7時に無人実行、Discord通知とコスト¥0を半年維持した運用ログ"
free: false
---

## ローカルcronとGitHub Actions無料枠、毎朝7時起動の2択を実行時間で比較

無人実行は2系統ある。常時起動PCがあるならcron、なければGitHub Actionsの無料2,000分/月で足りる。本Botは1回約90秒、月30回で45分しか消費せず、半年でも270分=無料枠の14%に収まった。

```bash
# ローカルcron: 毎朝7:00 JST にBot起動
0 7 * * * cd /home/bot/x-mute && /usr/bin/python3 main.py >> run.log 2>&1
```

```yaml
# .github/workflows/mute.yml — UTC指定なのでJST7時は22:00前日
on:
  schedule:
    - cron: "0 22 * * *"
jobs:
  mute:
    runs-on: ubuntu-latest
    timeout-minutes: 5
```

## GitHub Secretsへのbearer token格納とX API v2の月1万投稿上限ガード

改悪後の無料枠は読み取り月1万投稿。Secretsにキーを置き、起動直後に当月消費数を読んで上限の90%(9,000)で自動停止させる。

```python
import os, json, sys
TOKEN = os.environ["X_BEARER"]      # GitHub Secrets経由
used = json.load(open("usage.json")).get("month_posts", 0)
if used >= 9000:
    print("近月上限: 9000投稿到達、本日スキップ")
    sys.exit(0)
```

## 実行結果と当日ミュート件数をDiscord Webhookへ1リクエストで通知

成否・処理投稿数・ミュート追加件数を1通にまとめる。失敗時もexit前に必ず叩く。

```python
import requests
def notify(ok, posts, muted):
    msg = f"{'✅' if ok else '🛑'} 取得{posts}件 / 新規ミュート{muted}語 / 残枠{9000-posts}"
    requests.post(os.environ["DISCORD_URL"], json={"content": msg}, timeout=10)
```

## トークン失効・上限超過・辞書暴走の3障害を時系列で復旧

半年で起きた実障害と対処。辞書暴走は1日で語彙が412→3,800に膨張しTLが空になった事故で、日次差分に上限を設けて封じた。

```python
# 障害3: ミュート辞書の暴走を日次+50語でクランプ
new_words = detect_mute_candidates(timeline)
delta = list(set(new_words) - set(load_dict()))[:50]   # 1日最大50語
if len(detect_mute_candidates(timeline)) > 500:
    notify(False, len(timeline), 0); sys.exit(1)        # 異常検知で停止
save_dict(load_dict() + delta)
```

| 障害 | 発生月 | 原因 | 対処 |
|---|---|---|---|
| トークン失効 | 2ヶ月目 | bearer 90日期限切れ | 失効検知→Discord警告+手動再発行 |
| 上限超過 | 4ヶ月目 | リツイート急増で1万投稿到達 | 9,000ガード追加 |
| 辞書暴走 | 5ヶ月目 | 候補語の無制限追加 | 日次+50語クランプ |

## API消費投稿数の月次推移と失敗時セルフリトライ実装

消費は安定して月6,000〜7,500投稿(上限の75%)で着地。失敗は指数バックオフで3回まで自動再試行し、半年の手動介入はトークン再発行の1回のみ。

```python
import time
for attempt in range(3):
    try:
        run_mute_cycle(); break
    except Exception as e:
        wait = 2 ** attempt * 30          # 30s, 60s, 120s
        notify(False, 0, 0)
        time.sleep(wait)
else:
    sys.exit(1)
```

毎朝7時起動・残枠監視・障害自己復旧の3点が揃えば、無料枠内コスト¥0のまま半年連続稼働が成立する。次章では蓄積したミュート辞書をエクスポートし、複数アカウントへ横展開する。
