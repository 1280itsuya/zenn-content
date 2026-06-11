---
title: "第3章 Claude Haikuでノイズ分類器を作る：1件¥0.02・精度92%までのプロンプト改善ログ"
free: false
---

## 4分類のカテゴリ定義を曖昧さゼロで書く

精度68%だった初版の失敗は「宣伝」と「煽り」の境界をモデル任せにした点にある。Claude Haiku（`claude-haiku-4-5`）は定義が曖昧だと多数派ラベルへ寄る。判定基準を1行ずつ明文化し、排他条件まで書く。

```text
- useful: 一次情報・数値・再現手順を含む。リンク先が技術文書/論文
- promo: 自社/他社製品の購入・登録を主目的とする。アフィリンク含む
- bait: 「衝撃」「9割が知らない」等の煽り語が主。中身が定義1に満たない
- noise: 上記いずれにも該当しない雑談・自動投稿
判定が2つに跨る場合は promo > bait > noise の優先で1つだけ選ぶ
```

## Few-shot 3件とJSON強制で68%→88%

定義だけでは「useful風のpromo」を取りこぼす。誤判定3件を反例として注入し、`tools`でJSON出力を強制した時点で88%に乗った。

```python
import anthropic
client = anthropic.Anthropic()

SYSTEM = open("category_def.txt").read()
SHOTS = [
  {"text":"新刊『AI副業』予約受付中→bit.ly/xx","label":"promo"},
  {"text":"GPT-4oのレイテンシをp50/p95で実測した結果","label":"useful"},
  {"text":"これ知らないと2026年マジでヤバい","label":"bait"},
]
def build_msgs(post):
    ex = "\n".join(f'IN:{s["text"]}\nOUT:{{"label":"{s["label"]}"}}' for s in SHOTS)
    return [{"role":"user","content":f"{ex}\nIN:{post}\nOUT:"}]
```

## 残り4%の誤判定ログを潰す

88%で残った誤りは皮肉ポスト（褒め言葉で実は批判）の bait 取りこぼしだった。SHOTSに皮肉の反例を1件足し92%へ。検証は手ラベル200件を回す。

```python
import json
def classify(post):
    r = client.messages.create(
        model="claude-haiku-4-5", max_tokens=20,
        system=SYSTEM, messages=build_msgs(post))
    return json.loads(r.content[0].text)["label"]

gold = [json.loads(l) for l in open("eval200.jsonl",encoding="utf-8")]
hit = sum(classify(g["text"])==g["label"] for g in gold)
print(f"acc={hit/len(gold):.3f}")  # -> 0.920
```

## 1件¥0.02の内訳とトークン実測

Haikuは入力$1/出力$5（per MTok）。1件は入力620・出力12トークン。¥150換算で `(620*1+12*5)/1e6*150≈¥0.10`。SYSTEMとSHOTSを `cache_control` で固定し2回目以降の入力を1/10にすると、実効¥0.02まで落ちる。

```python
system=[{"type":"text","text":SYSTEM,
         "cache_control":{"type":"ephemeral"}}]
```

## 100件を1バッチでコスト1/4

定義+反例の固定部はキャッシュ済でも1件ずつ送ると往復オーバーヘッドが残る。100件を番号付きで1リクエストに束ね、配列で受け取ると合計トークンが約1/4になる。

```python
def classify_batch(posts):
    body="\n".join(f"{i}:{p}" for i,p in enumerate(posts))
    r=client.messages.create(model="claude-haiku-4-5",max_tokens=900,
        system=system,messages=[{"role":"user",
        "content":f"各行をJSON配列で分類:\n{body}"}])
    return json.loads(r.content[0].text)
```

切替判断は単純で、200件評価でHaikuが90%を割り、かつ誤分類が売上直結カテゴリ（promo誤検出）に偏った時のみ `claude-sonnet-4-6` へ。Sonnetは精度+3%だがコスト約12倍なので、噛むのは要約段（第4章）だけに限定する。
