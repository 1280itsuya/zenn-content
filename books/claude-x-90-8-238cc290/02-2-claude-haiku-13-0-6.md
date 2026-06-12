---
title: "第2章 Claude Haikuでツイートを13カテゴリ分類：プロンプト設計とトークン単価$0.6/日の最適化"
free: false
---

## SonnetからHaikuへ：1日$4を$0.6に下げた判断基準

Claude Sonnet 4.5で分類すると1,200ツイート/日で$4.1、月$123。Haiku 4.5に切り替えて$0.6/日（月$18）まで圧縮した。代償は誤分類率の悪化で、実測トレードオフは以下。

| モデル | 誤分類率 | 入力単価/MTok | 1日コスト |
|---|---|---|---|
| Sonnet 4.5 | 1.8% | $3.00 | $4.10 |
| Haiku 4.5 | 4.2% | $0.80 | $0.61 |

ノイズ除去用途では4.2%の取りこぼしは許容範囲、$3.5/日の差が決め手になった。

## 100件をJSON一括分類してAPI往復を1/100にする

1ツイート1リクエストだと1,200往復で429を踏む。100件を1配列でまとめ、`temperature=0`固定で出力を決定的にする。

```python
import anthropic, json
client = anthropic.Anthropic()

def classify_batch(tweets: list[str]) -> list[dict]:
    numbered = "\n".join(f"{i}: {t}" for i, t in enumerate(tweets))
    resp = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=2000, temperature=0,
        system=SYSTEM_PROMPT,
        messages=[{"role": "user", "content": numbered}],
    )
    return json.loads(resp.content[0].text)
```

往復は1/100、レイテンシも90秒→3秒に落ちた。

## 13カテゴリのenum制約で出力崩れを0件にする

カテゴリ名をプロンプト内でenum列挙し、「列挙外を返すな」と明示。これで2万件処理して未定義ラベルは0件。

```python
SYSTEM_PROMPT = """各ツイートを次の13カテゴリのいずれか1つに分類。
[tech, ai_news, money, spam_dm, stealth_ad, politics,
 gossip, recruit, giveaway, crypto_pump, adult, bot, other]
出力は {"idx":int,"cat":str,"noise":bool} の配列のみ。
列挙外のcatは禁止。説明文・コードフェンスを付けるな。"""
```

`noise:true`が除去対象。spam_dm/stealth_ad/crypto_pump/giveaway/botを自動でtrueに倒す。

## スパムDMとステマを見抜くFew-shot（15件中3件）

判定が割れるのはステマ（stealth_ad）と通常の感想。Few-shotで境界を固定する。

```text
"このサーバー有料级の情報量、DM開放してます" -> spam_dm / noise
"PR | 提供 @brandX 使ってみたら最高だった✨ 詳細はリンク" -> stealth_ad / noise
"普通にこの本良かった、3章のバックオフ実装が効いた" -> tech / keep
```

「PR」「提供」「DM開放」を含むがハッシュタグ単独では倒さない、という15件を同梱した。

## 429を指数バックオフで捌くリトライ

バッチ化しても深夜のまとめ処理で`RateLimitError`は出る。最大5回、2の指数で待つ。

```python
import time
from anthropic import RateLimitError

def with_retry(fn, *a, max_try=5):
    for n in range(max_try):
        try:
            return fn(*a)
        except RateLimitError:
            wait = 2 ** n          # 1,2,4,8,16秒
            time.sleep(wait)
    raise RuntimeError("429 exhausted after 5 tries")
```

実運用半年でこのリトライが拾った429は月平均7回、全て5回以内で成功した。

## 月$18に収めた使用量ダッシュボード

`usage`を毎リクエスト加算してJSONに追記、日次でしきい値$0.8を超えたらアラート。

```python
import json, pathlib
LOG = pathlib.Path("usage.jsonl")

def track(resp, day: str):
    u = resp.usage
    cost = u.input_tokens/1e6*0.8 + u.output_tokens/1e6*4.0
    LOG.open("a").write(json.dumps({"day": day, "cost": cost}) + "\n")
    total = sum(json.loads(l)["cost"]
               for l in LOG.read_text().splitlines()
               if json.loads(l)["day"] == day)
    if total > 0.8:
        print(f"ALERT {day}: ${total:.2f} > $0.8")
```

このログで月$18を一度も超えていない。3アカウント分を合算しても$54、Sonnet運用なら$369だった。
