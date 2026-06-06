---
title: "第5章 cron+Discord Webhookで毎朝7時に自動収集し失敗を通知する常駐化"
free: false
---

## cron で毎朝7時に probe.py を起動する3行

Linux なら `crontab -e` に1行追加すれば常駐は完了する。仮想環境の Python を絶対パスで指定し、stdout と stderr を日付付きログへ流す。

```bash
# crontab -e に追記（毎朝07:00 JST 実行）
0 7 * * * cd /home/itsuya/zenn-probe && /home/itsuya/zenn-probe/.venv/bin/python probe.py >> logs/probe_$(date +\%Y\%m\%d).log 2>&1
```

`%` は crontab 上で改行扱いになるため `\%` でエスケープする。この1点を忘れると07:00に何も起きず、原因究明に半日溶ける。

## Windows タスクスケジューラを schtasks で登録する

自宅PC（Windows 11）運用なら GUI を開かず `schtasks` で登録する。`/ru` に実行ユーザーを明示しないとログオフ中に発火しない。

```powershell
schtasks /Create /TN "ZennProbe" /SC DAILY /ST 07:00 `
  /TR "C:\zenn-probe\.venv\Scripts\python.exe C:\zenn-probe\probe.py" `
  /RU "ityya" /RP * /F
```

## .env で Cookie パスを外出しして失効に備える

Zenn の認証 Cookie は約30日で失効する。パスを `.env` に逃がし、ファイルの更新時刻が14日を超えたら警告を出す。

```python
import os, time
from pathlib import Path
from dotenv import load_dotenv

load_dotenv()
cookie_path = Path(os.environ["ZENN_COOKIE_PATH"])
age_days = (time.time() - cookie_path.stat().st_mtime) / 86400
if age_days > 14:
    raise RuntimeError(f"Cookie が{age_days:.0f}日経過。再ログインして書き出し直す")
```

## Discord Webhook へ取得件数と失敗理由を飛ばす

実行ごとに「成功:取得件数」か「失敗:例外名と先頭200字」を Discord に投げる。これで端末を開かずに07:01の結果を確認できる。

```python
import requests, traceback

def notify(content: str):
    requests.post(os.environ["DISCORD_WEBHOOK_URL"],
                  json={"content": content[:1900]}, timeout=10)

try:
    n = run_probe()           # 第4章の差分蓄積を実行
    notify(f"✅ Zenn収集 成功 / {n}記事のPVを保存")
except Exception:
    notify(f"❌ Zenn収集 失敗\n```{traceback.format_exc()[:200]}```")
    raise
```

## 60日運用で起きた3障害と復旧の時系列

放置運用といっても無故障ではない。60日で発生した障害は実測で3種・計5回。すべて Discord 通知が一次検知のトリガーになった。

| 日付 | 障害 | 通知内容 | 復旧 |
|---|---|---|---|
| Day12 | Cookie失効 | `401 Unauthorized` | 再ログインで Cookie 書き直し（5分） |
| Day31 | スキーマ変更 | `KeyError: 'total_count'` | キーが `page_views` 配下へ移動、パーサ1行修正 |
| Day31 | 同上 | 同上 | 既存JSONの後方互換 fallback 追加 |
| Day47 | 429 | `429 Too Many Requests` | 下記の指数バックオフで自動復旧 |

429 は再実行で抜けることが多いため、収集本体に retry を仕込んでおく。

```python
import time, requests

def get_with_backoff(url, headers, tries=4):
    for i in range(tries):
        r = requests.get(url, headers=headers, timeout=15)
        if r.status_code != 429:
            r.raise_for_status()
            return r.json()
        wait = 2 ** i          # 1,2,4,8秒
        time.sleep(wait)
    raise RuntimeError("429 が4回連続。当日はスキップし翌07:00へ")
```

Day31 のスキーマ変更だけは無人復旧できず手修正が要った。逆に言えば、60日のうち手を入れたのは1回・所要15分。`KeyError` を即通知する設計が、放置運用と「気づいたらデータが2週間欠損」を分ける唯一の境界線になる。
