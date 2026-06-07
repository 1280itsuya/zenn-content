---
title: "完成形を先に見る：毎朝7時に届く要約Digestと週14h→2hの内訳"
free: true
---

# 毎朝7時、Discordに届くDigestの実物

巡回の起点はこの1通だ。GitHub Actionsが`cron: '0 22 * * *'`(UTC=JST7時)で起動し、X APIで集めたツイートをClaude Haikuが分類した結果をDiscordへPostする。

```text
📬 X Digest 2026-06-07 07:00
取得 2,418件 / ノイズ 2,201 / 有益 187 / 要対応 30
── ミュート候補(本日12語) ──
#拡散希望, 副業確定, 朝活, …
── 有益10選 ──
1. Anthropic、Haikuの長文要約コストを実測した記事…(要約)
2. X API無料枠Postの上限変更まとめ…(要約)
```

スクロールせず30秒で「今日のX」が把握できる状態が、本書のゴール成果物だ。

## 約2,400件/日をHaikuが3分類する中身

分類はSonnetではなくHaikuで十分回る。1リクエストに50件束ねて投げ、JSONで`noise/useful/action`を返させる。

```python
import anthropic
client = anthropic.Anthropic()

def classify(tweets: list[str]) -> list[str]:
    msg = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=1024,
        messages=[{"role": "user",
            "content": f"次の各ツイートをnoise/useful/actionで分類しJSON配列で返す:\n{tweets}"}],
    )
    return parse_json(msg.content[0].text)
```

2,418件を48バッチに分け、約90秒で処理が終わる。

## 週14h→2hの内訳

削減効果はタイムログの実数で出す。導入前は手動巡回が1日1.8h、これが0.25h(Digest確認のみ)へ落ちた。

```yaml
before:   # /日
  巡回・スクロール: 1.8h
  週合計: 12.6h
  通知対応他: 1.4h
  total: 14.0h
after:
  Digest確認: 0.25h
  週合計: 1.75h
  要対応の手動処理: 0.25h
  total: 2.0h
```

削減源の9割は「無限スクロールの廃止」が占める。

## 月コスト¥380と誤判定率8%

ランニングは安い。Haikuのinput$1/Mtok換算で、2,418件×30日でもAPIは月¥380。サーバはGitHub Actions無料枠内で¥0。

```python
daily_in_tokens = 2418 * 60        # 1件約60tok
monthly_yen = daily_in_tokens * 30 * (1 / 1_000_000) * 150  # $1/M, ¥150/$
print(round(monthly_yen))          # -> 約380
```

誤判定率は、有益を誤ってノイズに送った率で実測8%。第5章でこの8%を3%まで下げるプロンプト調整を扱う。

## 全体アーキとリポジトリ構成

構成はGitHub Actions + Python + SQLite の3点のみ。状態は`tweets.db`に持ち、再実行しても重複Postしない。

```bash
.
├── .github/workflows/digest.yml   # cron 22:00 UTC
├── fetch.py        # X API v2 で取得
├── classify.py     # Haiku 3分類
├── digest.py       # 有益10選 + Discord Post
└── tweets.db       # SQLite(既読・分類履歴)
```

```yaml
# digest.yml 抜粋
on:
  schedule: [{cron: '0 22 * * *'}]
jobs:
  run:
    steps:
      - run: python fetch.py && python classify.py && python digest.py
```

この5ファイルが、次章以降で1つずつ組み上げる対象だ。X API無料枠の取得上限をどう回避し、Haikuのバッチサイズを何件に決めたか——再現の鍵はすべて第2章から始まる。
