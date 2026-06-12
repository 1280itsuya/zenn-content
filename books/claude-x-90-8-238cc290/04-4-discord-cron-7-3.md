---
title: "第4章 収集→要約→Discord配信の自動化：cronで毎朝7時に回す失敗3回ぶんのリカバリ設計"
free: false
---

## パイプライン全体を1関数に collect→classify→summarize→deliver で繋ぐ

第2章の取得と第3章の分類を `run_once()` に統合する。各ツイートを独立処理にし、1件の例外で全体を止めない。

```python
# pipeline.py
import logging
from collect import fetch_timeline      # 第2章
from classify import is_useful          # 第3章
from summarize import summarize_claude
from deliver import post_discord

def run_once():
    tweets = fetch_timeline(limit=200)
    sent = 0
    for tw in tweets:
        try:
            if not is_useful(tw):        # 約200件→有益12件に絞る
                continue
            summary = summarize_claude(tw["text"])
            post_discord(summary, tw["url"])
            sent += 1
        except Exception as e:
            logging.exception("skip %s: %s", tw["id"], e)
            continue                     # 1件失敗で全停止させない
    return sent
```

朝の200件処理でClaude Haiku 3.5を使うと、入力約180kトークンで月額換算 約¥420。GPT-4o miniより要約の自分語り混入が少なく、後述の検証で再試行が減る。

## 障害1：深夜2時のX APIタイムアウトをヘルスチェックで握りつぶす

半年で4回、深夜帯に取得が `ReadTimeout` で落ち、cronが非ゼロ終了して翌朝の配信が無言停止した。指数バックオフで3回まで再試行する。

```python
import time, requests

def fetch_with_retry(fn, tries=3):
    for i in range(tries):
        try:
            return fn()
        except (requests.Timeout, requests.ConnectionError):
            wait = 2 ** i * 5          # 5s, 10s, 20s
            time.sleep(wait)
    raise RuntimeError("fetch failed after 3 tries")
```

3回リトライ導入後、API由来の配信欠落はゼロになった。

## 障害2：Claude要約に混じる「私が思うに」をバリデーションで弾く

要約に一人称や前置きが混ざると配信が読みにくい。プロンプトで禁止し、出力側でも正規表現で検査して1回だけ再生成する。

```python
import re, anthropic
client = anthropic.Anthropic()
BAN = re.compile(r"(私が|思います|まとめると|興味深い)")

def summarize_claude(text, retry=1):
    msg = client.messages.create(
        model="claude-haiku-3-5",
        max_tokens=200,
        system="ツイートを事実のみ3行で要約。一人称・感想・前置き禁止。",
        messages=[{"role": "user", "content": text}],
    )
    out = msg.content[0].text.strip()
    if BAN.search(out) and retry > 0:
        return summarize_claude(text, retry - 1)
    return out
```

禁止語検査で再生成率は約6%、それでも残る場合はそのまま流して人手で削る運用にした。

## 障害3：cookie失効の無言停止を失敗時Discord通知で可視化する

Playwright取得のcookieが約30日で失効し、取得0件のまま正常終了して気づくのに2日かかった。0件や例外を必ずDiscordへ通知する。

```python
def deliver_or_alert(webhook):
    sent = run_once()
    if sent == 0:
        requests.post(webhook, json={
            "content": "⚠️ 配信0件。cookie失効の可能性。再ログインを実行せよ"
        })
```

BAN率は3アカウント半年で2回（約0.4%/日換算）。通知導入後、失効の検知は平均2日→当朝に短縮した。

## Windows Task Scheduler / cron 登録とロックファイルで二重起動を防ぐ

二重起動で同じ要約が2回配信される事故を、`filelock` で排他する。

```python
from filelock import FileLock, Timeout
try:
    with FileLock("/tmp/x_pipe.lock", timeout=1):
        deliver_or_alert(WEBHOOK)
except Timeout:
    pass   # 既に実行中なら静かに終了
```

```bash
# Linux: 毎朝7時
0 7 * * * /usr/bin/python3 /opt/x/pipeline.py >> /var/log/x.log 2>&1
```

```powershell
# Windows: 毎朝7時に登録
schtasks /Create /SC DAILY /ST 07:00 /TN "XPipe" `
  /TR "python C:\x\pipeline.py"
```

この5部品で、90分の手動巡回が毎朝7時の8分自動配信に置き換わった。
