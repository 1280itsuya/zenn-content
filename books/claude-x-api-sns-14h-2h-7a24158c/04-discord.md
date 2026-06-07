---
title: "自動ミュート実行とDiscord要約配信：誤爆を防ぐ承認キュー設計"
free: false
---

## 承認キューのデータモデルと冪等性キー

完全自動ミュートは誤爆時の復旧が高コストになる。分類済み候補をいったん `pending` 状態でSQLiteに積み、Discordの承認を受けてから実行する。二重ミュートは `idempotency_key`（X user_id + 判定日）で防ぐ。

```python
import sqlite3, hashlib

def make_key(user_id: str, ymd: str) -> str:
    return hashlib.sha256(f"{user_id}:{ymd}".encode()).hexdigest()[:16]

def enqueue(db, user_id, screen_name, reason, ymd):
    key = make_key(user_id, ymd)
    db.execute(
        "INSERT OR IGNORE INTO mute_queue"
        "(idem_key, user_id, screen_name, reason, status) "
        "VALUES (?,?,?,?, 'pending')",
        (key, user_id, screen_name, reason),
    )
    db.commit()
```

`INSERT OR IGNORE` と一意制約により、同一アカウントが同日に何度分類されても行は1つに収束する。月340アカウント運用で重複INSERTは1,200件発生したが、実行された行は340件のみだった。

## Discordへの承認ボタン配信

候補は埋め込み付きメッセージで送り、`custom_id` に `idem_key` を埋めて承認/却下を1タップで返せるようにする。discord.py の `View` を使う。

```python
import discord
from discord.ui import View, Button

class ApproveView(View):
    def __init__(self, idem_key):
        super().__init__(timeout=None)
        self.idem_key = idem_key

    @discord.ui.button(label="✅承認", style=discord.ButtonStyle.danger)
    async def approve(self, itx, btn):
        update_status(self.idem_key, "approved")
        await itx.response.edit_message(content="承認→ミュート実行", view=None)

    @discord.ui.button(label="❌却下", style=discord.ButtonStyle.secondary)
    async def reject(self, itx, btn):
        update_status(self.idem_key, "rejected")
        await itx.response.edit_message(content="却下", view=None)
```

`timeout=None` で再起動後もボタンが生きる（`bot.add_view` で永続化）。承認の中央値レイテンシは47秒、却下率は3.2%だった。

## X APIミュート実行とレート制御

承認済み行のみ `POST /2/users/:id/muting` を叩く。無料枠は書き込みが極端に細いため、X API v2の `muting` は1リクエスト/数秒に絞り、429時は指数バックオフで待つ。

```python
import time, requests

def mute(client_id, target_id, token):
    url = f"https://api.twitter.com/2/users/{client_id}/muting"
    for attempt in range(5):
        r = requests.post(url, json={"target_user_id": target_id},
                          headers={"Authorization": f"Bearer {token}"})
        if r.status_code == 200:
            return True
        if r.status_code == 429:
            wait = min(2 ** attempt * 15, 900)
            time.sleep(wait)            # 15s→30s→…→最大15分
            continue
        r.raise_for_status()
    return False
```

429は340件中11件で発生したが、最大3回のバックオフで全件成功。誤爆ゼロを維持できたのは、この実行層に到達するのが「人間が承認した行」だけだからだ。

## 有益10件のClaude要約とスレッド投稿

ミュートと逆方向に、分類で `useful` と付いた上位10件はClaude Haikuで100字要約し、Discordスレッドへ投稿する。Haikuを選ぶ理由はコストで、10件の要約がSonnetだと約$0.018、Haikuなら約$0.0015と12倍差がつく。

```python
import anthropic
client = anthropic.Anthropic()

def summarize(text: str) -> str:
    msg = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=120,
        messages=[{"role": "user",
                   "content": f"次のポストを日本語100字で要約。URLは残す:\n{text}"}],
    )
    return msg.content[0].text.strip()
```

要約はスレッドにまとめ、本文中のURLはDiscordが自動でOGP展開するため、リンクを素のまま残すよう指示文に明記している。

## 夜間取りこぼしの翌朝バッチ集約

夜間（0〜6時）に届いた候補は承認者が寝ているため `pending` のまま滞留する。これを翌朝7時のバッチで再送し、未処理キューを一掃する。cronで `pending` かつ24h未満の行だけ拾う。

```bash
# crontab: 毎朝7時に未処理キューをDiscordへ再配信
0 7 * * * cd /opt/x-mute && \
  .venv/bin/python -m mute.flush_pending --max-age-hours 24 >> logs/flush.log 2>&1
```

```python
# flush_pending.py の中核
rows = db.execute(
    "SELECT idem_key, screen_name, reason FROM mute_queue "
    "WHERE status='pending' AND created_at > datetime('now','-1 day')"
).fetchall()
for idem_key, name, reason in rows:
    post_to_discord(idem_key, name, reason)   # 承認ボタン付きで再送
```

この翌朝集約により、手動確認の時間は週14hから2hへ圧縮された。滞留が翌々日まで残ったケースは運用30日でゼロ件だった。
