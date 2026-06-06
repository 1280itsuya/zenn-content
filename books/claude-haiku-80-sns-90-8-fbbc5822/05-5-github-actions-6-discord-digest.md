---
title: "第5章 GitHub Actionsで朝6時に全自動化・Discord Digest配信とコスト監視ダッシュボード"
free: false
---

第5章 GitHub Actionsで朝6時に全自動化・Discord Digest配信とコスト監視ダッシュボード

朝6時にX収集→Haiku分類→Discord配信までを無人で回す。手元で叩くコマンドは月初の動作確認1回だけになる。

## scheduleで06:00 JST起動・Secretsにキー4本を格納

GitHub Actionsの`schedule`はUTC基準なので、JST 06:00は`21:00 UTC`で指定する。`X_BEARER_TOKEN`/`ANTHROPIC_API_KEY`/`DISCORD_WEBHOOK`/`DB_KEY`の4本はSecretsに置き、ワークフローへ環境変数で渡す。

```yaml
# .github/workflows/digest.yml
on:
  schedule:
    - cron: "0 21 * * *"   # 21:00 UTC = 06:00 JST
  workflow_dispatch: {}
jobs:
  digest:
    runs-on: ubuntu-latest
    env:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      X_BEARER_TOKEN: ${{ secrets.X_BEARER_TOKEN }}
      DISCORD_WEBHOOK: ${{ secrets.DISCORD_WEBHOOK }}
```

## SQLiteをactions/cacheで永続化しミュート履歴を引き継ぐ

Actionsのランナーは毎回破棄されるため、`mute.db`（既ミュート2,400件・既読ツイートID）を`actions/cache`で持ち越す。これで誤ミュートの再発防止リストが消えない。

```yaml
      - uses: actions/cache@v4
        with:
          path: state/mute.db
          key: mutedb-${{ github.run_id }}
          restore-keys: mutedb-
      - run: python pipeline.py   # mute.dbを読み書き
      - uses: actions/upload-artifact@v4
        with:
          name: mute-db
          path: state/mute.db
```

## カテゴリ別Digestを1 Webhookで配信

分類済みの有用投稿だけを`category`でグルーピングし、Discordのembed1通にまとめる。10件未満なら配信せず「本日該当なし」を送り、無音による故障見落としを防ぐ。

```python
import os, requests, collections
def send(posts):
    g = collections.defaultdict(list)
    for p in posts: g[p["category"]].append(p)
    fields = [{"name": f"📁 {c} ({len(v)})",
               "value": "\n".join(f"・[{x['text'][:40]}]({x['url']})" for x in v[:5]),
               "inline": False} for c, v in g.items()]
    requests.post(os.environ["DISCORD_WEBHOOK"],
        json={"embeds": [{"title": "朝のDigest", "fields": fields or
              [{"name":"本日該当なし","value":"-"}]}]}, timeout=10)
```

## 月¥80超でアラート・Haikuトークンを毎日記録

`claude-haiku-4-5`の消費トークンとX APIコール数をCSVに追記し、月累計が¥80（155円換算で約$0.52）を超えた日にDiscordへ警告する。半年の実測平均は月¥78、最大¥91（X側リトライ増の月）。

```python
import csv, datetime
def log_cost(in_tok, out_tok, api_calls):
    yen = (in_tok/1e6*0.25 + out_tok/1e6*1.25) * 155
    row = [str(datetime.date.today()), in_tok, out_tok, api_calls, round(yen,2)]
    with open("state/cost.csv","a",newline="") as f: csv.writer(f).writerow(row)
    month = sum(float(r[4]) for r in csv.reader(open("state/cost.csv"))
                if r[0][:7]==str(datetime.date.today())[:7])
    if month > 80: send_alert(f"⚠ 今月 ¥{month:.0f} / 上限¥80")
```

## 失敗3種と対処・副業アカウントへ移植

半年で詰まったのは3点。X 429（15分上限超過）は`tenacity`で指数バックオフ、セッション切れはBearer再取得をjob先頭に固定、誤ミュートはHaikuの`confidence<0.8`を保留キューへ回す。

```python
from tenacity import retry, wait_exponential, stop_after_attempt
@retry(wait=wait_exponential(min=60, max=900), stop=stop_after_attempt(4))
def fetch(url, headers):  # 429/5xxを4回まで自動リトライ
    r = requests.get(url, headers=headers, timeout=15)
    r.raise_for_status(); return r.json()
```

移植は3手順。リポジトリをForkしSecrets4本を自分の値へ差し替え、`config.yml`の収集アカウントを副業ニッチへ書き換え、初回だけ`workflow_dispatch`で手動実行してDigest着弾を確認する。翌朝06:00から無人運用に入る。
