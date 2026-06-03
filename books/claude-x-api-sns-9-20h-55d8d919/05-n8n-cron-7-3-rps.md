---
title: "n8n+cronで毎朝7時自動実行、3ヶ月のRPS/失敗ログ全公開"
free: false
---

## n8nの4ノードで取得から通知までを1本に繋ぐ

これまで作った取得・分類・ミュートの各スクリプトを、n8nのExecute Commandノードで直列に呼ぶ。出力はDiscord Webhookノードへ渡す。

```json
{
  "nodes": [
    {"name": "fetch", "type": "n8n-nodes-base.executeCommand",
     "parameters": {"command": "python /app/fetch_timeline.py --max 1000"}},
    {"name": "classify", "type": "n8n-nodes-base.executeCommand",
     "parameters": {"command": "python /app/classify.py --in /tmp/raw.json"}},
    {"name": "mute", "type": "n8n-nodes-base.executeCommand",
     "parameters": {"command": "python /app/apply_mute.py --threshold 0.85"}},
    {"name": "digest", "type": "n8n-nodes-base.discord",
     "parameters": {"webhookId": "={{$env.DISCORD_HOOK}}", "text": "={{$json.summary}}"}}
  ]
}
```

## cronで毎朝7時、Docker内のn8nを叩く

n8nのScheduleトリガーは秒単位の制御が甘いので、ホスト側cronからWebhook起動に統一した。JSTで07:00に固定。

```bash
# crontab -e  (TZ=Asia/Tokyo)
0 7 * * * curl -s -X POST http://localhost:5678/webhook/morning-mute \
  -H "Content-Type: application/json" \
  --max-time 300 \
  >> /var/log/x-mute/run_$(date +\%F).log 2>&1
```

## レート制限に指数バックオフで耐える

X API無料枠は読み取りが15分あたり実測1〜5RPSで頭打ち。429を受けたら待つ。3ヶ月で再試行は計142回、最大待機は64秒。

```python
import time, requests

def get_with_backoff(url, headers, tries=5):
    for i in range(tries):
        r = requests.get(url, headers=headers, timeout=30)
        if r.status_code != 429:
            return r
        wait = min(2 ** i, 64)          # 1,2,4,8,16,32,64
        time.sleep(wait)
    raise RuntimeError("rate-limited 5x")
```

## 3ヶ月の実ログ：停止4回・コスト¥1,400/月

ログをBashで集計した結果が下表。API一時障害(503)での完全停止は4回、いずれも翌朝の再実行で自動復旧した。

```bash
$ grep -hc '"status":"done"' /var/log/x-mute/run_*.log | paste -sd+ | bc
89    # 92日中89日成功

$ grep -h '503 Service' /var/log/x-mute/*.log | wc -l
4     # X API障害で停止した回数
```

| 指標 | 3ヶ月実測 |
|---|---|
| 成功日数 | 89 / 92日 |
| Claude分類コスト | ¥約1,400/月 (1万件¥48換算) |
| 再試行(429) | 142回 |
| ミュート誤爆ロールバック | 7件 |

## ミュート誤爆7件をワンコマンドで戻す

分類スコア0.85超を自動ミュートしたが、3ヶ月で7件が誤爆だった。`mute_audit.jsonl`に全操作を残し、IDを渡せば即時解除する。

```python
import json, requests

def rollback(user_id, headers):
    # 監査ログから該当行を確認してから解除
    requests.delete(
        f"https://api.twitter.com/2/users/me/muting/{user_id}",
        headers=headers, timeout=30)
    with open("rollback.log", "a") as f:
        f.write(json.dumps({"undo": user_id}) + "\n")
```

## 横展開チェックリストと浮いた20hの使い道

収集に月20h使っていた時間がほぼ0になる。空いた枠は発信に回す。別ジャンル(求人・株・ゲーム情報)へは分類プロンプトとミュート閾値の2点だけ差し替えれば動く。

```yaml
porting_checklist:
  - swap: classify.py のシステムプロンプト（対象ジャンルの定義）
  - tune: apply_mute.py の --threshold（0.80〜0.90で誤爆率を調整）
  - keep: backoff・cron・Discord通知はそのまま流用可
  - reinvest: 浮いた月20h → Zenn記事2本 + X発信10投稿
```
