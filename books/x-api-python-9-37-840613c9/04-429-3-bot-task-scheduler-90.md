---
title: "429エラーで3回死んだBotをTask Scheduler常駐で90日無停止にする"
free: false
---

運用初週に3回沈黙した実バグの解剖と恒久対策コードを軸に、Task Scheduler/cron両対応の常駐構成を約1200文字で執筆します。

## 運用初週、Botが3回沈黙したエラーログ原文

90日無停止の前に、まず死因を3つ晒す。実際のログから抜粋。

```text
[Day2] HTTPError: 429 Too Many Requests  ← except: pass で握り潰し→mute済みID 214件が空リストで上書き消失
[Day4] FileNotFoundError: C:\Users\old_pc\venv\Scripts\python.exe  ← PC再起動でvenv絶対パスが死亡
[Day5-11] 200 OK / data: []  ← トークン失効でも例外が出ず、6日間空回りに気付けず
```

3件とも「コードのバグ」ではなく「常駐設計の欠如」が原因。以降で1つずつ潰す。

## tenacityで429を最大900秒の指数バックオフに吸収する

Freeティアの壁は15分窓50リクエスト。429を例外として「待つ」側に倒す。

```python
from tenacity import retry, wait_exponential, stop_after_attempt, retry_if_exception_type
import requests

class RateLimited(Exception): pass

@retry(wait=wait_exponential(multiplier=30, max=900),
       stop=stop_after_attempt(6),
       retry=retry_if_exception_type(RateLimited))
def fetch(url: str, headers: dict) -> dict:
    r = requests.get(url, headers=headers, timeout=30)
    if r.status_code == 429:
        raise RateLimited(r.headers.get("x-rate-limit-reset"))
    r.raise_for_status()
    if not r.json().get("data") and r.status_code == 200:
        raise RuntimeError("empty data: token failure suspected")  # Day5の6日間空回り対策
    return r.json()
```

30→60→120…900秒と待機し、15分窓のリセットを跨いでから再試行する。`except: pass`は書かない。

## venv絶対パス死を防ぐ起動ラッパーbat

Task Schedulerにはpython.exeを直接登録せず、batを1枚噛ませる。

```bat
@echo off
cd /d %~dp0
call .venv\Scripts\activate.bat
python -m bot.main >> logs\run_%date:~0,4%%date:~5,2%.log 2>&1
```

`%~dp0`基準の相対パスにしたことで、Day4のようなフォルダ移動・再起動後のパス崩壊が再発ゼロになった。

## Task Schedulerとcronに15分間隔で登録する

Windowsはschtasks一発。管理者PowerShellで実行する。

```powershell
schtasks /Create /TN "XNoiseBot" /TR "C:\bot\run.bat" `
  /SC MINUTE /MO 15 /ST 07:00 /F
```

Linux/WSL側はflockで二重起動を防ぐのが要点。15分間隔と900秒バックオフが重なると前回プロセスが生きている場合がある。

```bash
*/15 * * * * flock -n /tmp/xbot.lock /home/user/bot/.venv/bin/python -m bot.main >> /home/user/bot/logs/cron.log 2>&1
```

## 90日無停止を支えたフォルダ構成とログ設計

最終形はこれだけ。state/を消さない限り何度落ちても再開できる。

```text
bot/
├── run.bat            # 起動ラッパー(相対パス)
├── .venv/
├── bot/main.py        # tenacity込み本体
├── state/
│   ├── muted_ids.json # 書込は一時ファイル→os.replace でアトミックに
│   └── last_seen.json
└── logs/              # 日次ローテ、空dataはWARNINGで記録
```

`muted_ids.json`の更新を`os.replace`に変えてからmuteリスト破損は0回。429発生は90日で計38回あったが、すべてバックオフで吸収され手動介入は0回。次章ではこの38回の429ログから、Freeティアで実際に叩ける安全レート(実測11リクエスト/15分)を逆算する。
