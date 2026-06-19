---
title: "記事生成パイプライン：Claude APIで1記事を90分→7分にした実装"
free: false
---

## 記事生成パイプライン：Claude APIで1記事を90分→7分にした実装

手作業で技術記事を1本書くと、テーマ選定・アウトライン・本文・校正で軽く90分かかる。これをパイプライン化して7分に落とした構成を解説する。

---

## プロンプトテンプレートの設計

最大の落とし穴は「大きなプロンプト1つで全部やらせる」設計だ。Claude は指示が多いと後半を無視し、タイトルと本文が全く違うテーマになるドリフトが起きる。

解決策は**役割分離**。プロンプトを3段階に分割する。

```python
PROMPTS = {
    "plan": """
テーマ: {topic}
このテーマで技術記事のアウトラインを JSON で返せ。
{"title": "...", "sections": ["...", "..."]}
タイトルと sections は必ず同じテーマを扱うこと。
""",
    "write": """
タイトル: {title}
構成: {sections}
上記タイトルと構成に沿って本文を書け。
タイトルのテーマから逸れた内容を書いてはいけない。
""",
    "validate": """
タイトル: {title}
本文: {body}
本文がタイトルのテーマと一致しているか確認し、
{"ok": true} か {"ok": false, "reason": "..."} を返せ。
"""
}
```

`plan → write → validate` の3ステップで、各プロンプトは単一責務に絞る。

---

## バッチ処理の実装

1記事ずつ同期で呼ぶと10記事で70分かかる。`asyncio` で並列化すると7分まで圧縮できる。

```python
import asyncio
import anthropic

client = anthropic.Anthropic()

async def generate_article(topic: str) -> dict:
    # Step 1: plan
    plan_resp = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=512,
        messages=[{"role": "user", "content": PROMPTS["plan"].format(topic=topic)}]
    )
    plan = json.loads(plan_resp.content[0].text)

    # Step 2: write
    write_resp = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        messages=[{"role": "user", "content": PROMPTS["write"].format(
            title=plan["title"], sections=plan["sections"]
        )}]
    )
    body = write_resp.content[0].text

    # Step 3: validate
    val_resp = client.messages.create(
        model="claude-haiku-4-5-20251001",  # 検証は軽量モデルで十分
        max_tokens=128,
        messages=[{"role": "user", "content": PROMPTS["validate"].format(
            title=plan["title"], body=body
        )}]
    )
    verdict = json.loads(val_resp.content[0].text)
    if not verdict["ok"]:
        raise ValueError(f"品質NG: {verdict['reason']}")

    return {"title": plan["title"], "body": body}


async def batch_generate(topics: list[str]) -> list[dict]:
    tasks = [generate_article(t) for t in topics]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return [r for r in results if not isinstance(r, Exception)]
```

**ポイント**: 検証ステップだけ `claude-haiku` を使う。Sonnet の1/5のコストで十分な精度が出る。

---

## 出力検証ロジック

バリデーションで見るべき項目は3つに絞る。

```python
def validate_output(title: str, body: str) -> bool:
    # 1. 文字数チェック（短すぎる記事を弾く）
    if len(body) < 600:
        return False

    # 2. タイトルキーワードの本文出現確認
    title_words = title.replace("｜", " ").split()[:3]
    if not any(w in body for w in title_words):
        return False

    # 3. LLM による意味整合チェック（上記コード参照）
    return llm_validate(title, body)
```

ルールベース（文字数・キーワード）を先に走らせてAPIコールを節約し、通過したものだけLLMに渡す。

---

## 実際の落とし穴

**JSON パース失敗**: Claude がコードブロック付きで返すことがある。

```python
import re

def extract_json(text: str) -> dict:
    # ```json ... ``` を除去してからパース
    text = re.sub(r"```json?\s*|\s*```", "", text).strip()
    return json.loads(text)
```

**レート制限**: Tier 1 は毎分50リクエストが上限。10記事×3ステップ=30リクエストなので、同時並列数は `asyncio.Semaphore(8)` で制御するのが安全だ。

このパイプラインで10記事/日を安定稼働させると、API コストは1記事あたり約¥8（Sonnet+Haiku 混在）に収まる。
