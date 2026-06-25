---
title: "第6章：viewデータを個人開発ループに還元する——勝ちテーマ抽出と次サイクル自動改善"
free: false
---

## 第6章：viewデータを個人開発ループに還元する——勝ちテーマ抽出と次サイクル自動改善

### なぜ「勝ちパターンの還元」が必要か

記事を量産しても、view数は記事ごとに大きくばらつく。原因を放置したまま同じプロンプトで回し続けると、同じ0view記事を量産するだけだ。解決策は、実測データからシグナルを抽出し、次の生成プロンプトへ自動で差し込む**閉じたフィードバックループ**を作ることである。

---

### winner_extractorの設計

`src/winner_extractor.py` は以下の3ステップで動く。

1. **ZennのAPIからview数上位N件を取得**
2. **タイトル・タグを集計して「勝ちパターン」を抽出**
3. **`data/winner_context.json`に書き出し、次サイクルのプロンプトが読み込む**

```python
import anthropic, json, requests

ZENN_API = "https://zenn.dev/api/articles?username=YOUR_USERNAME&order=liked_count"

def fetch_top_articles(n=10):
    res = requests.get(ZENN_API, timeout=10)
    articles = res.json().get("articles", [])
    # view数でソート（Zenn APIはliked_countのみ公開のためproxy指標として使用）
    return sorted(articles, key=lambda a: a.get("liked_count", 0), reverse=True)[:n]

def extract_winners(articles):
    client = anthropic.Anthropic()
    titles = [a["title"] for a in articles]
    tags   = [t["name"] for a in articles for t in a.get("topics", [])]

    prompt = f"""
以下は実際にview/likeを獲得したZenn記事のタイトルとタグです。
タイトル: {titles}
タグ: {tags}

これらに共通するテーマ・言葉の傾向・効果的なタグを3点ずつ日本語で箇条書きにしてください。
出力はJSON: {{"themes": [...], "tags": [...], "title_patterns": [...]}}
"""
    msg = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=512,
        messages=[{"role": "user", "content": prompt}]
    )
    return json.loads(msg.content[0].text)

def run():
    articles = fetch_top_articles()
    winners  = extract_winners(articles)
    with open("data/winner_context.json", "w", encoding="utf-8") as f:
        json.dump(winners, f, ensure_ascii=False, indent=2)
    print("winner_context.json を更新しました")

if __name__ == "__main__":
    run()
```

---

### 生成プロンプトへの差し込み方

`data/winner_context.json` を毎朝の記事生成エージェントが読み込み、プロンプトに埋め込む。

```python
import json

def load_winner_context():
    try:
        with open("data/winner_context.json", encoding="utf-8") as f:
            return json.load(f)
    except FileNotFoundError:
        return {}   # 初回は空でスキップ

def build_article_prompt(topic: str) -> str:
    ctx = load_winner_context()
    hint = ""
    if ctx:
        hint = f"""
【勝ちパターンのヒント（実viewデータより）】
- 有効テーマ: {ctx.get('themes', [])}
- 効果的タグ: {ctx.get('tags', [])}
- タイトルパターン: {ctx.get('title_patterns', [])}
これらを参考にしつつ、テーマは「{topic}」から外れないこと。
"""
    return f"{hint}\n{topic}についてZenn向け技術記事を書いてください。"
```

**ポイント：** ヒントはあくまで「参考」として渡す。強制すると、テーマと無関係なタグや別テーマのタイトルパターンが混入し、記事の一貫性が崩れる（筆者が実際に踏んだ失敗）。

---

### 落とし穴：LLMは数値を捏造する

winner_extractorの出力を「何件中何位」のような数値付きで生成させると、LLMが架空のview数を創作する。**実測値はAPIから取得し、LLMには傾向の解釈だけを依頼する**のが鉄則だ。数値分析はPythonで行い、LLMの仕事は「パターンの言語化」に限定する。

---

### スケジューリング

`winner_extractor.py` はTask Schedulerで週1回（月曜7:05）に実行する。毎日走らせるとサンプルが蓄積する前にノイズが入りやすい。

```
Trigger: 毎週月曜 07:05
Action : python C:\path\to\src\winner_extractor.py
```

これで「先週の勝ちパターン → 今週の記事生成プロンプト」という1週間サイクルの自動改善ループが完成する。
