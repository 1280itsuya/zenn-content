---
title: "第5章 GitHub Actionsで毎朝7時に無料常駐させDiscordへPVサマリを通知する完全自動化"
free: false
---

## cronで毎朝7:00 JST (UTC 22:00) に起動するworkflow定義

GitHub Actionsの`schedule`はUTC基準なので、JST 7:00は`cron: '0 22 * * *'`と書く。第3章で作った`probe.py`をそのまま呼び出す。

```yaml
# .github/workflows/zenn-probe.yml
name: zenn-pv-probe
on:
  schedule:
    - cron: '0 22 * * *'   # JST 07:00
  workflow_dispatch:        # 手動実行も残す
jobs:
  probe:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - run: pip install httpx
      - run: python probe.py
        env:
          ZENN_COOKIE: ${{ secrets.ZENN_COOKIE }}
```

実測では1ジョブの所要は約40秒。無料枠2000分なら毎日実行しても月20分の消費で、残り1980分は他リポジトリに回せる。

## ZENN_COOKIEをRepository Secretsへ格納し429を回避する

第4章で抜いた非公開統計JSON用のCookieは、平文でコミットせずSecretsに入れる。Networkタブで確認した`connect.sid`を含む文字列を丸ごと登録する。

```bash
gh secret set ZENN_COOKIE --body "connect.sid=s%3Axxxx...; _zenn_session=yyyy..."
```

probe側は`os.environ`から読み、429を踏んだら`Retry-After`を尊重して待つ。

```python
import os, time, httpx
cookie = os.environ["ZENN_COOKIE"]
r = httpx.get(url, headers={"Cookie": cookie}, timeout=10)
if r.status_code == 429:
    wait = int(r.headers.get("Retry-After", 30))
    time.sleep(wait)        # 実測30秒で解除
    r = httpx.get(url, headers={"Cookie": cookie})
```

GitHub Runnerの共有IPは複数ユーザーで429を誘発しやすく、記事30本を直列取得する場合は各リクエスト間に0.5秒のsleepを挟むと429がほぼゼロになった。

## SQLite永続化はartifactとcommitのどちらを選ぶか

収集結果`stats.db`を翌日に引き継ぐ方法は2つ。artifactは90日で消えるが履歴が汚れない。commit方式は無期限だがdiffが毎日増える。

```yaml
# 方式A: artifact（保持90日・容量制限500MB）
      - uses: actions/upload-artifact@v4
        with: { name: stats-db, path: stats.db, retention-days: 90 }
# 方式B: commit（無期限・リポジトリに常駐）
      - run: |
          git config user.name bot
          git add stats.db
          git commit -m "stats $(date +%F)" && git push || true
```

PV差分を1年追う本probeでは方式Bを推奨。`stats.db`は1日あたり約4KB増で、365日でも1.5MB程度に収まる。

## Discord WebhookへPV増加トップ5を整形通知する

前日比でPV増加が大きい順に5件をソートし、Webhookへ投げる。これで手動集計が月0分になる。

```python
import os, httpx, sqlite3
con = sqlite3.connect("stats.db")
rows = con.execute("""
  SELECT title, pv - prev_pv AS diff FROM today
  ORDER BY diff DESC LIMIT 5
""").fetchall()
lines = [f"{i+1}. {t} +{d}PV" for i,(t,d) in enumerate(rows)]
httpx.post(os.environ["DISCORD_WEBHOOK"],
           json={"content": "**本日のPV増加TOP5**\n" + "\n".join(lines)})
```

毎朝7:01に通知が届き、伸びた記事だけを即追記できる。文字数2000超で分割が要るDiscord仕様も、トップ5に絞れば1メッセージに収まる。

## このprobe基盤をnote・はてなへ横展開する3拡張点

同じ構成は媒体を差し替えるだけで再利用できる。変更点は3つに限定される。

```python
PROBES = {
  "zenn":   {"url": "https://zenn.dev/api/...",   "cookie": "ZENN_COOKIE"},
  "note":   {"url": "https://note.com/api/v2/...", "cookie": "NOTE_COOKIE"},
  "hatena": {"url": "https://blog.hatena.ne.jp/...","cookie": "HATENA_COOKIE"},
}
for name, cfg in PROBES.items():
    fetch_and_store(name, cfg)   # テーブルにsource列を足すだけ
```

(1)各媒体のNetworkタブで非公開JSONのエンドポイントを特定、(2)Secretsにcookieを追加、(3)`source`列で出自を区別する。3媒体を1ジョブにまとめても所要は約90秒で、無料枠2000分には全く届かない。
