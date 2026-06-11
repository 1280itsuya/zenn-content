---
title: "第1章 完成デモ：毎朝7時にDiscordへ届く『ノイズ除去済みTL要約』を5分で再現"
free: true
---

# 第1章 完成デモ：毎朝7時にDiscordへ届く『ノイズ除去済みTL要約』を5分で再現

## 30秒で動かす：3行の.envとclone

リポジトリ `x-noise-mute` をcloneし、`.env` に3つ書くだけで起動する。設定はXのBearer TokenとClaude APIキー、Discord WebhookのURLの3行だけ。

```bash
git clone https://github.com/example/x-noise-mute && cd x-noise-mute
cp .env.example .env   # 下記3行を編集
# X_BEARER_TOKEN=...
# ANTHROPIC_API_KEY=...
# DISCORD_WEBHOOK_URL=...
uv sync && uv run python -m app.run --once   # 即時1回実行
```

`--once` で即座にTLを取得→判定→要約→Discord投稿まで通す。初回の成功を確認してからcron常駐に移す。

## Claude Haikuがノイズを弾く：判定コア

TLの各ポストを `claude-haiku-4-5` で keep / mute に二値分類する。プロンプトに「宣伝・バズ狙い・無関係な日常」をmute条件として渡し、JSONで返させる。

```python
import anthropic, json
client = anthropic.Anthropic()

def judge(posts: list[str]) -> list[bool]:
    msg = client.messages.create(
        model="claude-haiku-4-5",
        max_tokens=1024,
        messages=[{"role": "user", "content":
            "次の各ポストをkeep/muteで判定しJSON配列で返す。"
            "mute=宣伝/バズ狙い/無関係な雑談。\n" + json.dumps(posts, ensure_ascii=False)}],
    )
    return [d["keep"] for d in json.loads(msg.content[0].text)]
```

300ポストの実測で、手動仕分けとの一致率は91.2%。残る誤判定の傾向は第4章の失敗ログで全件公開する。

## 毎朝7時に投げる：cron 1行

常駐は外部スケジューラに任せず、OSのcronに `--once` を1行登録するだけにした。プロセスが寝ている時間はメモリを食わない。

```bash
# crontab -e
0 7 * * * cd /home/itsuya/x-noise-mute && /usr/bin/uv run python -m app.run --once >> log/run.log 2>&1
```

Windowsならタスクスケジューラで同じコマンドを毎日07:00に登録する。`run.log` には取得件数・mute件数・所要秒数が1行ずつ追記され、後述の工数ログの原資料になる。

## Discordに届く実物：Webhook整形

要約はClaudeが生成した3〜5行のダイジェストを、Discordの埋め込みとして送る。スクリーンショットの通り、毎朝7:00ちょうどに通知が1件届く。

```python
import requests, os
def post_discord(summary: str, kept: int, muted: int):
    requests.post(os.environ["DISCORD_WEBHOOK_URL"], json={
        "embeds": [{
            "title": f"☀️ 今朝のTL要約（{kept}件採用 / {muted}件ミュート）",
            "description": summary,
            "color": 0x1DA1F2,
        }]
    }, timeout=10)
```

## 工数とコスト：90分→5分、月¥587の明細

導入前は朝にTLを遡って90分。現在は届いた要約を読む5分だけで、判断はcronとClaudeが肩代わりする。31日間のAPI課金は以下の実測。

```text
# 2026-05 利用実績（app/cost_report.py 出力）
Claude Haiku  入力 2,140,000 tok / 出力 96,000 tok  → $3.78
X API (Free枠 内 / 月10,000読取)                    → $0.00
合計  $3.78 ≒ ¥587  （目標 ¥600/月 以内）
```

X無料API枠の月10,000読取に収める取得設計が、この¥587を成立させている。枠を超えた日の挙動と回避策は次章で扱う。

ここまでが完成品の全体像で、cloneすれば今日から毎朝の要約が届く。第2章からは、この判定プロンプトの精度を自分のフォロー対象に合わせて引き上げる手順と、X無料枠を超えずに取得を回すレート設計を、実コードと失敗ログ付きで分解していく。
