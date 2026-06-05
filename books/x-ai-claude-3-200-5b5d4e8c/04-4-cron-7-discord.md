---
title: "第4章 cronで毎朝7時に回す常駐化と二重実行回避・Discord通知"
free: false
---

第4章は配管そのもの——「判定→投入」を毎朝7時に無人で回す常駐レイヤーを全文掲載する。以下、本文。

---

この章を読み終えると、`mute_runner.py` をWindowsタスクスケジューラ/cronに載せ、二重起動を防ぎ、cookie失効を検知してDiscordへ結果通知するところまで動く。完成すると毎朝の手作業はゼロになり、月次でミュート傾向レポートが自動生成される。

## ロックファイルで二重起動を防ぐ Python 実装

タスクスケジューラとcronを両方登録すると同時起動で二重ミュートが起きる。`portalocker` で排他ロックを取り、取れなければ即終了する。

```python
import portalocker, sys
from pathlib import Path

LOCK = Path("run.lock")
def acquire():
    fh = open(LOCK, "w")
    try:
        portalocker.lock(fh, portalocker.LOCK_EX | portalocker.LOCK_NB)
        return fh
    except portalocker.LockException:
        sys.exit("already running")  # 二重起動を黙って打ち切る
```

## 指数バックオフでPlaywright投入を3回まで再試行

X側の429やネットワーク断で1回失敗しただけで全停止しないよう、`2 ** attempt` 秒で最大3回待つ。

```python
import time
def with_backoff(fn, tries=3):
    for attempt in range(tries):
        try:
            return fn()
        except Exception as e:
            wait = 2 ** attempt  # 1s, 2s, 4s
            time.sleep(wait)
            last = e
    raise last
```

## cookie失効を検知してDiscordへ通知する

深夜にcookieが失効し、ログイン画面のまま30回空振りして通知欄が埋まった。ミュートボタンのセレクタが0件なら「失効」と断定し、空振り通知を1回に集約する。

```python
def notify(webhook, msg):
    import requests
    requests.post(webhook, json={"content": msg}, timeout=10)

count = page.locator('[data-testid="mute"]').count()
if count == 0:
    notify(WEBHOOK, "⚠️ cookie失効の可能性。再ログインが必要")
    sys.exit(1)
```

## 結果サマリ（新規ミュート数・削減率・APIコスト）を1通で送る

成功時はDiscordに定量サマリを送る。Claude Haiku判定のトークン実費も併記し、1日あたり約¥4で回っていることを可視化する。

```python
summary = (
    f"✅ 新規ミュート {new_muted} 件 / "
    f"推定削減率 {reduction:.0%} / "
    f"Claude API ¥{cost_yen:.1f}"
)
notify(WEBHOOK, summary)  # 例: 新規ミュート 18件 / 削減率 31% / ¥3.9
```

## タスクスケジューラに毎朝7時で常駐登録する

`schtasks` で7:00トリガーを作る。`/RL HIGHEST` を付けないとPlaywrightのヘッドレス起動が黙って失敗する。

```bash
schtasks /Create /TN "x-automute" /TR "C:\proj\.venv\Scripts\python.exe C:\proj\mute_runner.py" /SC DAILY /ST 07:00 /RL HIGHEST
```

## 月次ミュート傾向レポートを自動生成する

毎日のログを `mute_log.csv` に追記し、月初にカテゴリ別集計を出して「何を無音化したか」を残す。

```python
import pandas as pd
df = pd.read_csv("mute_log.csv", parse_dates=["date"])
m = df[df.date.dt.month == df.date.max().month]
print(m.groupby("category")["count"].sum().sort_values(ascending=False))
# spam 142 / 投資勧誘 89 / バズ便乗 53
```

これで判定から通知までが無人化され、月3,200語規模のノイズが毎朝自動で消える。次章ではこの常駐ジョブの誤ミュート率を1週間ログから逆算し、Claudeのプロンプト側へフィードバックする。

---

topics: `claude`, `anthropic`, `python`, `automation`, `api`
