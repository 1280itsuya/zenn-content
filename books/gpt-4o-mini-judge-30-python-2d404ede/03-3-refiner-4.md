---
title: "第3章 しきい値リトライとRefinerプロンプト設計：無限ループ・コスト爆発を防ぐ4ガード"
free: false
---

## max_retries=3と改善幅0.3で無限ループを断つ制御フロー

judgeの指摘を放置するとrefinerは同じ欠陥を再生産し続ける。打ち切り条件は「回数」と「改善幅」の2軸で持つ。前回スコアから0.3未満しか伸びなければ、それ以上回しても頭打ちなので即停止する。

```python
def write_loop(topic, judge, refiner, threshold=8.0, max_retries=3):
    draft = refiner.first_draft(topic)
    prev = 0.0
    for i in range(max_retries):
        score, critique = judge.score(draft)   # GPT-4o-mini judge
        if score >= threshold:
            return {"draft": draft, "score": score, "retries": i}
        if score - prev < 0.3:                  # 改善幅ガード
            return {"draft": draft, "score": score, "retries": i, "stop": "no_gain"}
        prev = score
        draft = refiner.revise(draft, critique)
    return {"draft": draft, "score": score, "retries": max_retries, "stop": "max"}
```

30記事中、`no_gain`で早期打ち切りされたのは7記事。これがコスト爆発を防ぐ最初のガードになる。

## 合格しきい値8.0とトークン上限：コスト¥0.4→¥1.5の内訳

しきい値8.0は、judge側にも上限を効かせないと意味がない。指摘文が長文化するとrefinerへの入力が膨らみ、1記事のコストが跳ねる。`max_tokens`をjudge=400・refiner=2000で固定する。

```python
JUDGE_MAX_TOKENS = 400
REFINER_MAX_TOKENS = 2000

def cost_yen(in_tok, out_tok):
    # gpt-4o-mini: $0.15/1M in, $0.60/1M out, 1USD=155円
    return ((in_tok * 0.15 + out_tok * 0.60) / 1_000_000) * 155
```

3回フルにリトライした最悪ケースで、judge×4回+refiner×3回=¥1.5。1発合格なら¥0.4。トークン上限なしの旧実装では指摘文が1200トークンまで伸び、最悪¥3.8まで膨らんでいた。

## Refinerに全文渡しvs要約渡し：スコア8.3 vs 7.6の実測

refinerへ「前回スコアと指摘文」をどう渡すかで最終品質が割れる。指摘を3行に要約して渡すと修正箇所が曖昧になり、全文をそのまま渡した方が伸びた。

```python
def revise(self, draft, critique, prev_score):
    msg = (
        f"前回スコア: {prev_score}/10。以下の指摘を全文反映し書き直す。\n"
        f"--- 指摘 ---\n{critique}\n"   # 要約せず全文
        f"--- 原稿 ---\n{draft}"
    )
    return self.client.responses(msg, max_tokens=REFINER_MAX_TOKENS)
```

30記事の最終スコア中央値は、全文渡し8.3に対し要約渡し7.6。差0.7は合格しきい値8.0をまたぐため、要約渡しだと合格率が47%→73%へ落ちた。トークン増分は1記事あたり+¥0.2で、品質差に対して安い。

## リトライしても上がらない記事の特徴と「捨てる」判断ロジック

`no_gain`で止まった7記事を分解すると失敗は2系統だった。「テーマが薄い」(具体例ゼロ・検索需要なし)が4本、「事実誤りを含む」(judgeが減点し続けるがrefinerが直せない)が3本。これらは回すほど赤字なので、入口で弾く。

```python
def should_drop(topic, first_score, critique):
    if first_score < 5.0:                       # 初回が低すぎる=テーマ薄い
        return True, "thin_topic"
    if any(k in critique for k in ("事実誤り", "誤情報", "出典不明")):
        return True, "factual_error"            # refinerで直らない系
    return False, None
```

初回スコア5.0未満をドロップ条件に入れると、無駄なリトライ7記事分=約¥7を削減でき、合格記事あたりの実コストを¥1.1から¥0.9へ下げられた。捨てる判断もコスト最適化の一部として実装する。
