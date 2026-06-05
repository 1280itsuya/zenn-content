---
title: "第3章 Claudeで投稿をノイズ判定：適合率92%まで詰めたプロンプト"
free: false
---

## 二段判定で課金を1/5に：Haiku一次→Sonnet再判定

全件を Sonnet に投げると高い。`claude-haiku-4-5` で全投稿を三値分類し、`confidence < 0.8` の境界例だけ `claude-sonnet-4-6` へ回す。実測で再判定に回るのは全体の14%だった。

```python
def classify(text: str):
    r = call_claude("claude-haiku-4-5", text)      # 一次
    if r["confidence"] < 0.8:
        r = call_claude("claude-sonnet-4-6", text) # 境界例のみ
    return r
```

## JSON強制とPydanticスキーマ検証

Claude にラベルだけ喋らせると表記揺れ（「ノイズ」「noise」）が出る。出力を JSON 固定し、`pydantic` で弾く。検証落ちは1回だけ再試行する。

```python
from pydantic import BaseModel
from typing import Literal

class Verdict(BaseModel):
    label: Literal["useful", "noise", "hold"]
    confidence: float
    reason: str

v = Verdict.model_validate_json(resp.content[0].text)
```

## few-shot例は「誤判定3件」を選ぶ

正例を並べても精度は伸びない。初版が外した実例（宣伝RT・ポエム・本題前置き）を few-shot に入れた瞬間、適合率が79%→92%へ跳ねた。

```python
FEWSHOT = [
  {"text": "【拡散希望】フォロー&RTで…", "label": "noise"},
  {"text": "朝活なう☕ がんばる", "label": "noise"},
  {"text": "X APIのv2、429後のRetry-Afterが", "label": "useful"},
]
```

## 200件の自前ラベルで68%→92%/再現率85%

評価セットは手で200件にラベル付けした。プロンプト差分ごとに `precision_score` を回し、回帰を検知する。

```python
from sklearn.metrics import precision_score, recall_score
print(precision_score(y, pred, pos_label="useful"))  # 0.68 → 0.92
print(recall_score(y, pred, pos_label="useful"))      # 0.85
```

## 1000投稿あたりの課金額を実測

Haiku 全件＋14%だけ Sonnet 再判定で、1投稿の平均入出力は約420トークン。1000投稿の請求実測は **¥41**（為替¥158/＄時）。

```bash
# usageログから1000件分のコストを集計
jq '[.[].usd] | add * 158' usage_2026-06.json
# => 41.3  (円/1000投稿)
```
