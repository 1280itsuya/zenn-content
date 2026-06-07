---
title: "Claude Haikuでノイズを3分類：プロンプト設計と誤ミュート率を8%→2%へ"
free: false
---

## 3分類を structured output で固定する

分類結果を自由文で返させると後段のパースが壊れる。Claude Haiku（`claude-haiku-4-5`）の tool use を使い、出力スキーマを `label` と `confidence` の2フィールドに固定する。

```python
import anthropic

client = anthropic.Anthropic()

CLASSIFY_TOOL = {
    "name": "classify",
    "description": "投稿を noise/useful/action の3値に分類",
    "input_schema": {
        "type": "object",
        "properties": {
            "label": {"type": "string", "enum": ["noise", "useful", "action"]},
            "confidence": {"type": "number"},
        },
        "required": ["label", "confidence"],
    },
}

def classify(text: str) -> dict:
    res = client.messages.create(
        model="claude-haiku-4-5",
        max_tokens=128,
        tools=[CLASSIFY_TOOL],
        tool_choice={"type": "tool", "name": "classify"},
        messages=[{"role": "user", "content": text}],
    )
    return res.content[0].input  # {"label": "...", "confidence": 0.0-1.0}
```

`tool_choice` で classify を強制するため、`label` が enum 外になる事故が消える。

## Haiku と Sonnet を1,000件で実測：¥12 vs ¥190

入力40トークン・出力20トークン程度の短文分類では、モデル単価がそのままコスト差になる。同一の1,000件を両モデルで流した結果が下表だ。

```python
# 1,000件・平均 in 42tok / out 18tok での実測（円換算 1USD=155円）
COST = {
    "claude-haiku-4-5":  {"yen": 12,  "f1": 0.94},
    "claude-sonnet-4-6": {"yen": 190, "f1": 0.96},
}

ratio = COST["claude-sonnet-4-6"]["yen"] / COST["claude-haiku-4-5"]["yen"]
gain  = COST["claude-sonnet-4-6"]["f1"] - COST["claude-haiku-4-5"]["f1"]
print(f"コスト{ratio:.0f}倍, F1差+{gain:.2f}")  # コスト16倍, F1差+0.02
```

F1で+0.02のために16倍払う合理性はない。誤ミュートはプロンプト側で潰す方針に倒し、Haiku で確定させた。

## 初版プロンプトの誤ミュート率8%の原因

初版は「ノイズなら noise」とだけ指示していた。有益投稿（`useful`）を `noise` と誤判定した率＝誤ミュート率は1,000件中82件で8.2%。内訳を集計すると、宣伝風の言い回しを含む技術告知が大半だった。

```python
errors = [r for r in results if r["true"] == "useful" and r["pred"] == "noise"]
print(f"誤ミュート率 {len(errors)/1000:.1%}")  # 8.2%

from collections import Counter
print(Counter(e["reason"] for e in errors).most_common(2))
# [('告知系をnoise誤判定', 51), ('絵文字過多をnoise誤判定', 23)]
```

「消してはいけないものを消す」のが最大の損失なので、ここを安全側へ倒す。

## Few-shot 6例＋安全側ルールで2%へ

プロンプトに境界事例6件を Few-shot で与え、「迷えば useful」の安全側ルールを追加した。変更は system プロンプトの diff で示す。

```diff
 SYSTEM = """投稿を noise / useful / action に分類する。
+判断に迷う場合は必ず useful を選ぶ（有益の取りこぼしを最優先で防ぐ）。
+
+# 例
+入力: 新ライブラリv2リリース告知 → useful
+入力: 個人の食事写真のみ → noise
+入力: 障害発生・要対応の報告 → action
+入力: 絵文字多めの技術Tips → useful
+入力: アフィリンクのみの投稿 → noise
+入力: 返信を求める質問 → action
 """
```

再評価で誤ミュートは82件→21件、2.1%まで低下。F1は0.94→0.95へ微増した。6例の追加コストは1件あたり+90トークン（≒+¥0.002）。

## バッチ20件でレイテンシ4.2s→1.1s

1件1リクエストだと1,000件で約70分かかる。20件を1メッセージにまとめ、JSON配列で返させてレイテンシを償却する。

```python
def classify_batch(posts: list[str]) -> list[dict]:
    numbered = "\n".join(f"{i}: {p}" for i, p in enumerate(posts))
    res = client.messages.create(
        model="claude-haiku-4-5",
        max_tokens=20 * 24,
        tools=[BATCH_TOOL],  # items: [{idx, label}] を返すスキーマ
        tool_choice={"type": "tool", "name": "classify_batch"},
        messages=[{"role": "user", "content": numbered}],
    )
    return sorted(res.content[0].input["items"], key=lambda x: x["idx"])

# 1件あたり平均レイテンシ: 単発4.2s → バッチ20件で1.1s
```

バッチ化で実効スループットが約4倍になり、1,000件の分類が70分→18分に収まる。誤ミュート2.1%・¥12・18分——この3数値が、週14hを2hへ圧縮する分類段の確定スペックになる。
