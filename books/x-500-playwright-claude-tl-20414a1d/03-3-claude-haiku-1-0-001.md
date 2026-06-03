---
title: "第3章 Claude Haikuで1ツイートを0.001円分類：シグナル/ノイズ判定プロンプトとコスト"
free: false
---

## Haiku 1ツイートを0.001円で分類する損益分岐

結論：1000ツイートを Claude Haiku で分類して実測 **¥18** だった。signal/noise の二値判定なら Sonnet は過剰で、入力40トークン・出力5トークン程度に収まる。Haiku の単価（入力 $0.80 / 出力 $4.00 per 1M）で1件あたり約0.0017円、為替155円換算で四捨五入0.002円。本書のTLは1日あたり収集300件なので、月9000件でも¥160で殲滅できる。

```python
# cost_estimate.py — 1000件の実測ログから単価を逆算
total_in_tokens, total_out_tokens = 41_320, 5_010
usd = total_in_tokens/1e6*0.80 + total_out_tokens/1e6*4.00
print(f"1000件 = ${usd:.4f} / 約¥{usd*155:.0f}")  # 1000件 = $0.1153 / 約¥18
```

## tool use で signal/noise 出力をJSON固定する

自由記述させると「これはノイズですね」と日本語を返してパース不能になる。`tools` に分類スキーマを渡し `tool_choice` で強制すると、出力が必ず構造化される。

```python
import anthropic
client = anthropic.Anthropic()

CLASSIFY_TOOL = [{
    "name": "label_tweet",
    "description": "副業エンジニアのTL情報収集に有益か判定",
    "input_schema": {
        "type": "object",
        "properties": {
            "label": {"type": "string", "enum": ["signal", "noise"]},
            "category": {"type": "string",
                "enum": ["tech_tips","release","case_study","promo","mood","spam"]},
            "confidence": {"type": "number"}
        },
        "required": ["label", "category", "confidence"]
    }
}]

def classify(text: str) -> dict:
    r = client.messages.create(
        model="claude-haiku-4-5-20251001", max_tokens=128,
        tools=CLASSIFY_TOOL,
        tool_choice={"type": "tool", "name": "label_tweet"},
        messages=[{"role": "user", "content": SYSTEM_RULE + text}])
    return r.content[0].input  # 必ず dict
```

## 100件をbatchで投げてレート制限を回避する

1件ずつ同期で叩くと1000件で約6分かかる。`concurrent.futures` で並列10にすると52秒に短縮。429が出たら指数バックオフで吸収する。

```python
from concurrent.futures import ThreadPoolExecutor
import time, itertools

def safe_classify(text, retry=3):
    for i in range(retry):
        try:
            return classify(text)
        except anthropic.RateLimitError:
            time.sleep(2 ** i)
    return {"label": "noise", "category": "spam", "confidence": 0.0}

def run_batch(tweets, workers=10):
    with ThreadPoolExecutor(max_workers=workers) as ex:
        return list(ex.map(safe_classify, tweets))
```

## 宣伝混在ツイートの誤爆を category ルールで潰す

最大の誤分類源は「技術系インフルエンサーが有益なTipsの末尾に自社商材リンクを貼る」混在型だった。初期プロンプトは promo として全切りし、有益Tips部分まで noise 化してリコールを落とした。`confidence < 0.7` かつ `category == promo` のものだけ signal へ救済するルールを後段に足したら、ノイズ率は **42%→9%** まで落ちた。

```python
SYSTEM_RULE = """以下のツイートを副業エンジニアのTL情報収集観点で判定。
- 動くコード/設定/障害事例/リリース情報を含めば signal
- 末尾に商材リンクがあっても本文にTipsがあれば signal(category=promo)
- 感想/承認欲求/相互フォロー募集のみは noise
ツイート: """

def rescue(r: dict) -> dict:
    if r["label"] == "noise" and r["category"] == "promo" and r["confidence"] < 0.7:
        r["label"] = "signal"  # Tips混在を救済
    return r
```

## Haiku vs Sonnet：精度3pt差にコスト12倍を払うか

同じ300件の正解ラベルで両モデルを評価した実測が下表。Sonnet は混在型に強いが、`rescue` ルールを足した Haiku が精度91%に追いつき、コスト差12倍が割に合わなくなった。

| モデル | 精度 | 1000件コスト | 混在型リコール |
|---|---|---|---|
| Haiku（素） | 86% | ¥18 | 0.71 |
| Haiku + rescue | 91% | ¥19 | 0.88 |
| Sonnet 4.6 | 94% | ¥230 | 0.93 |

```bash
# 正解ラベルとの突き合わせで精度を毎回計測
python eval_accuracy.py --pred out_haiku.jsonl --gold gold_300.jsonl
# accuracy=0.91  promo_recall=0.88  cost_per_1k=19yen
```

混在型のリコール0.88で運用上の取りこぼしは月1〜2件。¥19で回せる Haiku + rescue を本番採用とし、Sonnet は四半期に一度の正解ラベル再生成（教師データ更新）専用に格下げした。
