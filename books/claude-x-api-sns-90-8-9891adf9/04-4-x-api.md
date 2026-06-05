---
title: "第4章 ノイズを自動ミュート：X APIミュート連携と誤爆を防ぐ閾値設計"
free: false
---

## 三重ガード：スコア閾値・連続ノイズ回数・ホワイトリスト

第3章のClaude判定スコア(0.0-1.0)を単独で使うと巻き込み事故が起きる。`score >= 0.85`・`連続ノイズ3回以上`・`ホワイトリスト不在`の3条件を全て満たした時だけミュート候補に上げる。

```python
WHITELIST = set(open("whitelist.txt").read().split())

def should_mute(user_id: str, score: float, history: dict) -> bool:
    if user_id in WHITELIST:
        return False
    if score < 0.85:
        history[user_id] = 0
        return False
    history[user_id] = history.get(user_id, 0) + 1
    return history[user_id] >= 3
```

連続回数で「たまたまノイズ的な1投稿」を弾く。これが誤爆率を後述の1.8%まで下げた主因。

## X API mute エンドポイントと429リトライ

`POST /2/users/:id/muting` は15分あたり50回の制限。429が返ったら`x-rate-limit-reset`(UNIX秒)まで待つ。

```python
import time, requests

def mute(my_id, target_id, token):
    url = f"https://api.twitter.com/2/users/{my_id}/muting"
    for _ in range(3):
        r = requests.post(url, json={"target_user_id": target_id},
                          headers={"Authorization": f"Bearer {token}"})
        if r.status_code == 429:
            reset = int(r.headers["x-rate-limit-reset"])
            time.sleep(max(reset - int(time.time()), 1))
            continue
        r.raise_for_status()
        return r.json()
    raise RuntimeError("mute failed after 3 retries")
```

50件/15分を超える日は無く、実測で待機発生は月3回・最大47秒だった。

## Discord承認を挟む半自動モード

完全自動が怖いうちは、ミュート前にDiscord Webhookへ候補を流し、人が👍したものだけ実行する半自動モードを使う。

```python
import requests

def request_approval(user, score, webhook):
    payload = {"content": f"🔇 mute候補: @{user} (score={score:.2f})\n"
                          f"承認は `!mute {user}` を返信"}
    requests.post(webhook, json=payload)
```

承認コマンドを拾うbot側で`should_mute`を再評価してから`mute()`を呼ぶ。最初の2週間はこのモードで運用し、誤爆ゼロを確認してから全自動に切り替えた。

## 運用1ヶ月の実測：誤爆率1.8%、ノイズ比率31%→9%

2026年5月の実測値。自動ミュート167件のうち、手動解除したのは3件(誤爆率 3/167 = 1.8%)。

```yaml
period: 2026-05-01 .. 2026-05-31
auto_muted: 167
manual_unmuted: 3          # 誤爆
false_positive_rate: 0.018
tl_noise_ratio_before: 0.31
tl_noise_ratio_after: 0.09  # -22pt
api_429_events: 3
```

TLのノイズ比率(直近100件中のノイズ投稿割合)は31%→9%へ。誤爆3件はいずれもスコア0.85-0.88の境界で、次節の閾値見直しに回した。

## 閾値0.85→0.88への調整と再現率トレードオフ

誤爆3件を分析し、閾値を0.88へ上げると誤爆は0件想定だが、再現率(取りこぼし)が落ちる。実データで両方を出して判断する。

```python
def eval_threshold(samples, th):
    tp = sum(1 for s in samples if s["label"]=="noise" and s["score"]>=th)
    fp = sum(1 for s in samples if s["label"]=="good"  and s["score"]>=th)
    fn = sum(1 for s in samples if s["label"]=="noise" and s["score"]< th)
    return {"precision": tp/(tp+fp), "recall": tp/(tp+fn)}

for th in (0.85, 0.88, 0.90):
    print(th, eval_threshold(samples, th))
# 0.85 {'precision': 0.982, 'recall': 0.94}
# 0.88 {'precision': 1.0,   'recall': 0.88}
# 0.90 {'precision': 1.0,   'recall': 0.79}
```

適合率1.0を取ると再現率が0.94→0.88へ6pt落ちる。運用では「誤爆ゼロ優先」で0.88を採用し、取りこぼしは連続ノイズ回数を2回に下げて補った。
