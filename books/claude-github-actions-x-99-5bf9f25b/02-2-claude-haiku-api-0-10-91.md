---
title: "第2章 Claude Haiku APIでツイートのノイズスコアを0〜10で判定するプロンプト設計と精度91%への改善履歴"
free: false
---

## ノイズ判定カテゴリ5分類の定義と初期精度64%の壁

「ノイズ」をLLMに渡す前に、人間側が明確に定義しなければならない。曖昧な指示は曖昧な出力を生む。以下の5カテゴリを定義した。

| カテゴリ | 定義 | 初期誤検知率 |
|---|---|---|
| PROMO | URLを含む宣伝・PR告知 | 23% |
| FLAME | 感情的な煽り・対立誘発 | 31% |
| MISINFO | 数値・事実の誤りを含む | 44% |
| LOWQ | 内容が薄い・反応乞い | 19% |
| VALUE | 技術・実体験・一次情報 | 8% |

最初のシステムプロンプトは「ノイズかどうか判定してください」の一文だった。手動ラベリングした200件との一致率は64%。特にMISINFOの誤検知率44%が致命的で、「Xは将来性がない」という意見をMISINFOと誤分類するケースが多発した。

```python
# 初期プロンプト（精度64%時点）
SYSTEM_PROMPT_V1 = """
ツイートがノイズかどうか判定してください。
ノイズならscore 6-10、価値あるならscore 0-5を返してください。
"""
```

この曖昧さが64%止まりの根本原因だった。

## JSON出力を強制するシステムプロンプト設計

Claude HaikuはJSONモードを明示しないと自然文で返す確率が高い。以下の3テクニックで出力を安定させた。

1. **`You must respond with valid JSON only.`を冒頭に置く**
2. **スキーマをJSONで例示する**（型ヒントより効果大）
3. **各フィールドの意味を1行で説明する**

```python
SYSTEM_PROMPT_V2 = """You must respond with valid JSON only. No explanation, no markdown.

Output format:
{
  "score": <integer 0-10>,
  "category": <"PROMO"|"FLAME"|"MISINFO"|"LOWQ"|"VALUE">,
  "reason": <string, max 20 chars in Japanese>
}

Scoring rules:
- 0-2: 一次情報・技術解説・実体験
- 3-5: 普通のツイート
- 6-8: 宣伝・煽り・内容が薄い
- 9-10: 明確な誤情報・スパム
"""
```

スキーマ例示の追加だけでJSONパースエラー率が18%→2%に低下した。

## few-shot例5件の選定基準と精度91%への改善履歴

few-shotは「典型例」ではなく「境界例」を選ぶ。精度が上がらない原因は、境界ケースへの対応不足だった。

選定基準：
- URLありでもVALUEになるツイート（技術解説）
- 感情語を含むがFLAMEではないもの（批判的レビュー）
- 数値を含むがMISINFOではないもの（実測報告）

```python
FEW_SHOT_EXAMPLES = [
    {
        "tweet": "Claudeのキャッシュ機能で月のAPI費用が43%削減できた。system promptにcache_controlを追加するだけ。",
        "output": {"score": 1, "category": "VALUE", "reason": "一次情報・具体数値あり"}
    },
    {
        "tweet": "【PR】このツールで副業月収10万！今すぐ登録 https://example.com",
        "output": {"score": 9, "category": "PROMO", "reason": "PR明示・誘導URL"}
    },
    {
        "tweet": "AIはもう終わり。みんなそう言ってる。",
        "output": {"score": 7, "category": "FLAME", "reason": "根拠なし煽り"}
    },
    {
        "tweet": "GPT-4の精度は100%です（公式発表）",
        "output": {"score": 10, "category": "MISINFO", "reason": "事実誤認・存在しない公式"}
    },
    {
        "tweet": "今日のランチ美味しかった😊",
        "output": {"score": 5, "category": "LOWQ", "reason": "技術無関係・薄い内容"}
    }
]
```

精度改善の全履歴：

| バージョン | 変更点 | 精度 |
|---|---|---|
| v1 | 初期プロンプト（1文） | 64% |
| v2 | JSON強制+スコアルール追加 | 74% |
| v3 | カテゴリ定義を詳細化 | 81% |
| v4 | few-shot 5件追加（典型例） | 88% |
| v5 | few-shotを境界例に差し替え | 91% |

## 100ツイートを1リクエストでバッチ処理するPython実装

Haiku APIのコストは入力トークン依存なので、1ツイートずつ送ると100倍の呼び出しオーバーヘッドが乗る。100ツイートを1リクエストにまとめる実装：

```python
import anthropic
import json
from typing import List

client = anthropic.Anthropic()

def build_few_shot_messages() -> list:
    messages = []
    for ex in FEW_SHOT_EXAMPLES:
        messages.append({"role": "user", "content": ex["tweet"]})
        messages.append({"role": "assistant", "content": json.dumps(ex["output"], ensure_ascii=False)})
    return messages

def score_tweets_batch(tweets: List[str]) -> List[dict]:
    numbered = "\n".join(f"[{i}] {tweet}" for i, tweet in enumerate(tweets))
    
    user_prompt = f"""以下の{len(tweets)}件のツイートを判定し、JSON配列で返せ。
インデックスは0から{len(tweets)-1}まで全件含めること。

{numbered}

Output:
[
  {{"index": 0, "score": ..., "category": "...", "reason": "..."}},
  ...
]"""

    response = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=4096,
        system=SYSTEM_PROMPT_V2,
        messages=[
            *build_few_shot_messages(),
            {"role": "user", "content": user_prompt}
        ]
    )
    
    return json.loads(response.content[0].text)

# 使用例
tweets = load_timeline_tweets(limit=100)
results = score_tweets_batch(tweets)
noise_ids = [r["index"] for r in results if r["score"] >= 6]
print(f"ノイズ判定: {len(noise_ids)}/{len(tweets)}件")
```

100ツイート処理の実測コスト：**¥0.8/バッチ**（Haiku）。1日1,000ツイートを処理しても月額**約¥240**に収まる。

## claude-haiku-4-5 vs claude-sonnet-4-6: コスト対精度の実測比較

同一プロンプト・同一テストセット200件で両モデルを比較した。

| モデル | 精度 | 100件コスト | P95レイテンシ | 推奨 |
|---|---|---|---|---|
| claude-haiku-4-5-20251001 | 91% | ¥0.8 | 3.2秒 | **通常運用** |
| claude-sonnet-4-6 | 94% | ¥12.0 | 6.8秒 | 精度優先の場合のみ |

差分の3%は主にMISINFO検出の境界ケースで発生。1日1,000ツイートを処理するなら月額差は**¥240 vs ¥3,600**になる。精度3%の差にその金額を払うかどうかが判断基準になる。

## 誤検知パターン上位5件の原因と `is_opinion` フラグによる修正

91%で残る9%の誤検知内訳：

| 誤検知パターン | 件数/200件 | 原因 |
|---|---|---|
| 意見をMISINFOと誤判定 | 6件 | 断言口調をそのままMISINFO扱い |
| 長文VALUEをFLAMEと誤判定 | 4件 | 感情語の検出が過敏 |
| URLなし宣伝を見逃す | 3件 | PROMO=URL前提の学習バイアス |
| 英語ツイートのスコア不安定 | 3件 | 日本語前提のプロンプト |
| 引用RTの本文を無視 | 2件 | 引用元テキストのみ参照 |

最大の原因（6件）を潰すために `is_opinion` フラグをJSONスキーマへ追加した。

```python
# v5.1: opinion検出を追加したスキーマ定義
SCHEMA_V5_1 = """
{
  "score": <integer 0-10>,
  "category": <"PROMO"|"FLAME"|"MISINFO"|"LOWQ"|"VALUE">,
  "is_opinion": <boolean, 事実主張でなく意見・感想の場合 true>,
  "reason": <string, max 20 chars>
}

Rule: is_opinion=true のとき、MISINFO の最大スコアを 7 に制限する。
"""

# パース後の後処理でスコアを補正
def apply_opinion_cap(result: dict) -> dict:
    if result.get("is_opinion") and result["category"] == "MISINFO":
        result["score"] = min(result["score"], 7)
    return result
```

この1フィールド追加でMISINFO誤検知が6件→1件に減り、最終精度が**91%→92.5%**に到達した。
