---
title: "Pythonで作るコンテンツ生成エンジン——Claude APIプロンプト設計と実コード全公開"
free: false
---

## 勝ちタグ選定ロジック

コンテンツ生成の出発点は「何を書くか」ではなく「どのタグが流入を生むか」の判断だ。Zennの `/api/articles?tag=python&order=weekly` などで週次ビュー数を取得し、記事数が少なく平均ビューが高いタグを優先する。

```python
import httpx, statistics

async def rank_tags(candidate_tags: list[str]) -> list[str]:
    scores = {}
    async with httpx.AsyncClient() as client:
        for tag in candidate_tags:
            r = await client.get(f"https://zenn.dev/api/articles?tag={tag}&order=weekly")
            articles = r.json().get("articles", [])
            if not articles:
                continue
            views = [a.get("page_views_count", 0) for a in articles]
            # ビュー中央値 ÷ sqrt(記事数) で「競合薄×反応あり」を選ぶ
            scores[tag] = statistics.median(views) / (len(articles) ** 0.5)
    return sorted(scores, key=scores.get, reverse=True)
```

**落とし穴**: 記事数ゼロのタグはスコアが∞になって誤選出される。`if not articles: continue` で除外を忘れずに。

---

## タイトル生成プロンプト

タイトルが弱いと本文が良くても埋もれる。Claude APIへは「具体的な数字・対象読者・得られる結果」を強制するシステムプロンプトを渡す。

```python
import anthropic

client = anthropic.Anthropic()

def generate_title(tag: str, topic: str) -> str:
    msg = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=200,
        system=(
            "あなたはZenn/Qiita向けタイトル専門家です。"
            "必ず数字・読者層・得られる結果を含む30文字以内のタイトルを1つだけ返してください。"
            "説明や前置きは一切不要です。"
        ),
        messages=[{"role": "user", "content": f"タグ: {tag}\nトピック: {topic}"}],
    )
    return msg.content[0].text.strip()
```

**実例**: `tag="python"`, `topic="非同期処理"` → *「asyncioで3倍速──Python非同期入門、5つの実パターン」*

---

## 本文生成プロンプト設計

本文は「タイトルと同一テーマを維持する」のが最大の難所だ。モデルはユーザーターンで新しい話題を振られると脱線する。テーマをシステムプロンプトに固定することで防ぐ。

```python
def generate_body(title: str, tag: str) -> str:
    system = (
        f"あなたは『{title}』という記事の執筆者です。"
        "このタイトルのテーマ以外には絶対に触れないこと。"
        "見出しはMarkdown ##、コードブロックは```python で書くこと。"
        "字数: 1200〜1800字。"
    )
    msg = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        system=system,
        messages=[{"role": "user", "content": f"タグ: {tag}\n上記タイトルの記事本文を書いてください。"}],
    )
    body = msg.content[0].text
    # ドリフト検出: タイトルのキーワードが本文に含まれているか確認
    keywords = set(title.replace("──", " ").split())
    if not any(kw in body for kw in list(keywords)[:3]):
        raise ValueError(f"本文がタイトルから逸脱しています: {title}")
    return body
```

**落とし穴**: `max_tokens=512` では途中で打ち切られ、コードブロックが閉じないまま終わる。本文生成は最低でも2048トークンを確保する。

---

## 非同期バッチ処理で1記事7分を実現する

タグ選定→タイトル→本文を直列で回すと1記事あたり20〜30秒かかる。`asyncio.gather` で並列化すると10記事を同時処理できる。

```python
import asyncio

async def produce_article(tag: str, topic: str) -> dict:
    title = generate_title(tag, topic)          # 同期でも可、必要なら to_thread へ
    body = generate_body(title, tag)
    return {"tag": tag, "title": title, "body": body}

async def batch_produce(topics: list[tuple[str, str]]) -> list[dict]:
    tasks = [produce_article(tag, topic) for tag, topic in topics]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    ok = [r for r in results if isinstance(r, dict)]
    ng = [r for r in results if isinstance(r, Exception)]
    if ng:
        print(f"[WARN] {len(ng)}件失敗: {ng[0]}")
    return ok

if __name__ == "__main__":
    topics = [
        ("python", "非同期処理"),
        ("claude", "プロンプト設計"),
        ("fastapi", "ミドルウェア実装"),
    ]
    articles = asyncio.run(batch_produce(topics))
    print(f"生成完了: {len(articles)}件")
```

`return_exceptions=True` を使うと1件の失敗でバッチ全体が止まらない。本番ではここにリトライ処理とコスト集計（`usage.input_tokens + usage.output_tokens`）を追加しておくと運用が安定する。

---

## まとめ

| ステップ | キーポイント |
|---|---|
| タグ選定 | 中央値ビュー ÷ √記事数 でスコアリング |
| タイトル生成 | 数字・読者・結果を強制するシステムプロンプト |
| 本文生成 | テーマ固定＋ドリフト検出で品質ゲート |
| バッチ処理 | `asyncio.gather` + `return_exceptions` で安定並列 |

次章では、生成した記事をZenn/Qiitaへ自動投稿するPoster層の実装に進む。
