---
title: "第4章 Batches APIとprompt cachingでトークン費用を52%削減する実装と実測ベンチ"
free: false
---

## Batches APIで非同期ジョブを投げ単価を50%にする

リアルタイム応答が不要な処理は Message Batches API へ寄せるだけで入力・出力ともに単価が半額になる。月3万件のうち約2.4万件（要約・分類・タグ付け）をバッチへ移し、ここだけで API 費用が ¥41,000 → ¥27,800 に落ちた。投入からポーリングまでの最小実装はこうだ。

```python
import anthropic, time
client = anthropic.Anthropic()

batch = client.messages.batches.create(requests=[
    {
        "custom_id": f"job-{i}",
        "params": {
            "model": "claude-haiku-4-5-20251001",
            "max_tokens": 512,
            "system": SYSTEM_PROMPT,         # 後述: cache対象
            "messages": [{"role": "user", "content": text}],
        },
    } for i, text in enumerate(docs)
])

while True:                                   # 完了まで60秒間隔ポーリング
    b = client.messages.batches.retrieve(batch.id)
    if b.processing_status == "ended":
        break
    time.sleep(60)
```

完了まで最長24hかかるが、夜間バッチ運用なら遅延は無視できる。実測の中央値は1万件あたり38分だった。

## cache_controlで共通システムプロンプトを82%ヒットさせる

24件中ほぼ全リクエストで共通する長いナレッジ（約3,200トークン）は、毎回フル課金すると無駄が大きい。`cache_control` を付けると2回目以降は cache_read 単価（通常入力の1/10）で済む。

```python
system=[{
    "type": "text",
    "text": KNOWLEDGE_BASE,                  # 3,200トークンの固定知識
    "cache_control": {"type": "ephemeral"},  # 5分TTLでキャッシュ
}]
```

注意点は cache_creation トークンが通常入力の1.25倍課金される点だ。ヒット率が低いと逆に高くつく。半年運用での失敗として、TTL5分を跨ぐまばらな投入でヒット率が31%まで落ち、1日あたり ¥480 を溶かした月があった。バッチを時間帯で束ねて連続投入する運用に変え、ヒット率82%まで戻している。

## cache_creation/cache_readの料金差を実データで読む

レスポンスの `usage` を必ずログに残す。これがないと改善は再現できない。

```python
u = result.message.usage
print(u.cache_creation_input_tokens,  # 初回のみ発生
      u.cache_read_input_tokens,      # ヒット分
      u.input_tokens)                 # 非キャッシュ入力
```

1万件あたりの実測内訳：

| 項目 | キャッシュ前 | キャッシュ後 |
|---|---|---|
| 通常入力トークン課金 | ¥6,200 | ¥2,400 |
| cache_read 課金 | ¥0 | ¥310 |
| 入力課金合計 | ¥6,200 | ¥2,710（**61%減**） |

## haiku-4-5とopus-4-8をタスク難度で振り分ける

全件を opus-4-8 で処理すると品質は高いが費用が3倍になる。文字数と要求精度で2モデルにルーティングし、品質スコア（人手評価サンプル200件）を落とさず単価だけ削った。

```python
def route(text: str) -> str:
    if len(text) > 4000 or needs_reasoning(text):
        return "claude-opus-4-8"            # 難タスク 約12%
    return "claude-haiku-4-5-20251001"      # 易タスク 約88%
```

opus へ回したのは全体の12%。この振り分けで全体品質スコアは91→90とほぼ維持しつつ、Batches・prompt caching・ルーティングの合算で月 **¥41,000 → ¥19,600（52%削減）** に到達した。内訳はバッチ化で約32%、cache で約14%、ルーティングで約6%が効いている。

技術書やプログラミングスクールでこの手の運用設計をさらに体系的に学びたい読者向けの教材は[A8.netのプログラミング講座](https://px.a8.net/)から探せる。
