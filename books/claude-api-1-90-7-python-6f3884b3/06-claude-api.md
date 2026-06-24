---
title: "実測データで見るClaude APIコスト最適化——キャッシュ・モデル選択・バッチ処理で月額を半減させる"
free: false
---

## 実測データで見るClaude APIコスト最適化——キャッシュ・モデル選択・バッチ処理で月額を半減させる

個人開発の自動化パイプラインでClaude APIを使い続けると、月末に請求額を見て驚くことがある。本章では実際の運用データをもとに、**モデル選択・Prompt Caching・バッチ設計**の3つの軸でコストを半分以下に落とした方法を具体的に示す。

---

## モデル選択：Haikuを使い分けるだけで8割減

2025年時点のAnthropicの料金体系（実測ベース）は以下の通りだ。

| モデル | 入力 $/1Mトークン | 出力 $/1Mトークン | 用途 |
|---|---|---|---|
| claude-opus-4-8 | $15.00 | $75.00 | 複雑な推論・最終判断 |
| claude-sonnet-4-6 | $3.00 | $15.00 | 汎用・中程度の複雑さ |
| claude-haiku-4-5 | $0.80 | $4.00 | 分類・要約・定型生成 |

ブログ記事を1日12本生成する場合、Sonnetで回すと月約$48かかっていたが、Haikuに切り替えて**月$9**になった。切り替え実装は1行だ。

```python
# config/agents.json
{
  "blog_writer": {
    "model": "claude-haiku-4-5-20251001",  # 変更前: claude-sonnet-4-6
    "max_tokens": 2000
  }
}
```

```python
import anthropic
import json

def get_model(agent_name: str) -> str:
    with open("config/agents.json") as f:
        return json.load(f)[agent_name]["model"]

client = anthropic.Anthropic()

response = client.messages.create(
    model=get_model("blog_writer"),
    max_tokens=2000,
    messages=[{"role": "user", "content": prompt}]
)
```

**落とし穴**：Haikuは長文の一貫性が弱い。2000字超の記事を一発生成すると後半でトーンが崩れることがある。対策は**章ごとの分割生成**（後述）。

---

## Prompt Caching：同じシステムプロンプトを繰り返すなら必須

Prompt Cachingを使うと、同一プレフィックスの入力トークンが**90%オフ**になる。記事生成のような「システムプロンプトが固定で、ユーザープロンプトだけ変わる」パターンに最適だ。

```python
def generate_with_cache(system_prompt: str, user_prompt: str) -> str:
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=2000,
        system=[
            {
                "type": "text",
                "text": system_prompt,
                "cache_control": {"type": "ephemeral"}  # キャッシュ対象を明示
            }
        ],
        messages=[{"role": "user", "content": user_prompt}]
    )
    
    # キャッシュヒット確認
    usage = response.usage
    print(f"cache_read: {usage.cache_read_input_tokens}, "
          f"cache_creation: {usage.cache_creation_input_tokens}")
    
    return response.content[0].text
```

実測での効果（システムプロンプト2000トークン、1日12回呼び出し）：

| | キャッシュなし | キャッシュあり |
|---|---|---|
| 日次入力トークン | 24,000 | 2,000（初回）+ 2,200×11 |
| 月額換算 | $0.58 | $0.07 |

**落とし穴**：キャッシュのTTLは**5分**。パイプラインを順番に実行していると5分を超えてキャッシュが失効し、毎回課金される。対策はバッチを一括実行するか、`cache_control`を`persistent`に設定する（執筆時点でベータ機能）。

---

## 3軸を組み合わせたコスト試算

私の実際のパイプライン（1日：ブログ12本・Zenn1本・Pinterest10件・要約3件）の月額変遷。

```
最適化前（全てSonnet、キャッシュなし）：約 $48/月
↓ モデルをHaikuに切替（複雑タスク以外）
→ 約 $12/月
↓ Prompt Cachingを有効化
→ 約 $7/月
↓ 夜間バッチ化（並列→直列化でAPI再試行削減）
→ 約 $5/月（▲約90%）
```

最終的にClaude Max（$100/月定額）に移行したが、API課金のままでも**月$5-10**の水準に収まる。

---

## 実装チェックリスト

- [ ] `config/agents.json` でモデルを一元管理し、タスクの複雑度でHaiku/Sonnetを使い分ける
- [ ] システムプロンプトが500トークン超なら `cache_control: ephemeral` を必ずつける
- [ ] `usage.cache_read_input_tokens` をログに記録し、キャッシュヒット率を週次で確認する
- [ ] 連続呼び出し間隔が5分を超えるパイプラインではキャッシュを期待しない設計にする

コスト最適化は一度設定すれば終わりではなく、**ログで実態を把握→ボトルネックを特定→設定を更新**のサイクルを月1回回すことが効く。次章では、このコスト削減効果を保ちながら生成品質を落とさないLLM-as-Judgeの実装を紹介する。
