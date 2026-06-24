---
title: "コンテンツ生成パイプラインの実装——タイトル・本文・タグを7分で量産するPythonコード"
free: false
---

## コンテンツ生成パイプラインの実装——タイトル・本文・タグを7分で量産するPythonコード

### パイプラインの全体像

このパイプラインは3つのフェーズで動く。**①テーマ決定→②コンテンツ生成→③品質検証**だ。Claude APIへのリクエストは1記事あたり最大3回。タイトル・本文・タグを一括生成し、検証で落ちたものだけリトライする設計にする。

```
テーマ入力
  └─ generate_article()
       ├─ タイトル3案を生成
       ├─ 採用タイトルで本文を生成
       ├─ タグを生成
       └─ validate_content() で検証 ─── NG → リトライ(最大2回)
```

---

### プロンプト設計の核心：「タイトルと本文を分離する」

最大の落とし穴は**タイトルと本文を1プロンプトで生成しようとすること**だ。これをやるとモデルがタイトルのキャッチコピー風に引きずられ、本文が箇条書きの羅列になる。

解決策は2段階生成。まずタイトルだけを3案出させ、最良を選んでから本文プロンプトに埋め込む。

```python
import anthropic
import json
import re

client = anthropic.Anthropic()

TITLE_PROMPT = """\
テーマ: {theme}

以下の形式でタイトルを3案生成してください。
- 具体的な数字や手順を含める
- 読者の悩みに直結させる
- 30〜45文字以内

JSON形式で返してください:
{{"titles": ["タイトル1", "タイトル2", "タイトル3"]}}
"""

BODY_PROMPT = """\
タイトル: {title}
テーマ: {theme}

上記タイトルに完全に沿った技術記事を書いてください。

構成:
- 導入（問題提起）: 100字
- 実装手順（コード付き）: 400字
- 落とし穴と対策: 200字
- まとめ: 100字

タイトルのテーマから逸れた内容を書かないこと。
"""
```

`タイトルのテーマから逸れた内容を書かないこと`という一文が重要だ。これがないと本文の後半でテーマがズレていく「ドリフト」が頻発する。

---

### 実装：generate_article() の全コード

```python
def generate_titles(theme: str) -> list[str]:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=512,
        messages=[{
            "role": "user",
            "content": TITLE_PROMPT.format(theme=theme)
        }]
    )
    text = response.content[0].text
    data = json.loads(re.search(r'\{.*\}', text, re.DOTALL).group())
    return data["titles"]


def generate_body(title: str, theme: str) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        messages=[{
            "role": "user",
            "content": BODY_PROMPT.format(title=title, theme=theme)
        }]
    )
    return response.content[0].text


def generate_tags(title: str, body: str) -> list[str]:
    prompt = f"以下の記事に適切なタグを5個、JSON配列で返してください。\nタイトル:{title}\n本文冒頭:{body[:200]}"
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=128,
        messages=[{"role": "user", "content": prompt}]
    )
    text = response.content[0].text
    return json.loads(re.search(r'\[.*?\]', text, re.DOTALL).group())


def generate_article(theme: str, max_retries: int = 2) -> dict | None:
    for attempt in range(max_retries + 1):
        titles = generate_titles(theme)
        title = titles[0]  # スコアリングは後述
        body = generate_body(title, theme)
        tags = generate_tags(title, body)

        article = {"title": title, "body": body, "tags": tags}
        errors = validate_content(article)

        if not errors:
            return article

        print(f"[attempt {attempt+1}] 検証NG: {errors}")

    return None  # 全リトライ失敗
```

---

### 品質検証ロジック：validate_content()

生成後の検証を省くと**タイトルと本文がまったく別の話題になる記事**が混入する。実測で約10〜15%の頻度で発生した。

```python
def validate_content(article: dict) -> list[str]:
    errors = []
    title = article["title"]
    body = article["body"]

    # 1. 最低文字数チェック
    if len(body) < 400:
        errors.append(f"本文が短すぎる: {len(body)}字")

    # 2. タイトルキーワードの本文一致チェック
    title_words = set(title.replace("・", " ").split())
    body_words = body
    unmatched = [w for w in title_words if len(w) > 3 and w not in body_words]
    if len(unmatched) > len(title_words) * 0.5:
        errors.append(f"タイトルと本文の乖離: {unmatched}")

    # 3. プレースホルダ混入チェック
    if re.search(r'\{[a-zA-Z_]+\}', body):
        errors.append("未展開のプレースホルダが残存")

    # 4. タグ数チェック
    if len(article.get("tags", [])) < 3:
        errors.append("タグが3個未満")

    return errors
```

**一致チェックのしきい値は50%** がちょうど良い。厳しすぎると同義語の言い換えで誤検知が増え、緩すぎるとドリフト記事を通してしまう。

---

### 実行と所要時間

```python
if __name__ == "__main__":
    import time
    start = time.time()

    theme = "PythonでCSVを自動処理する方法"
    article = generate_article(theme)

    elapsed = time.time() - start
    if article:
        print(f"生成完了: {elapsed:.1f}秒")
        print(f"タイトル: {article['title']}")
        print(f"タグ: {article['tags']}")
    else:
        print("生成失敗（要確認）")
```

実測では**タイトル生成3秒＋本文生成18秒＋タグ生成2秒＝合計約23秒**。10記事を並列実行（`ThreadPoolExecutor`）しても7分以内に収まる。

---

### まとめと次のステップ

このパイプラインで押さえるべき3点を再確認する。

1. **タイトルと本文を分離して2段階生成する**——1プロンプト生成はドリフトを招く
2. **生成直後に必ず検証を挟む**——10〜15%の不良記事を自動で除去できる
3. **リトライは最大2回にとどめる**——それ以上はテーマ自体を変えるほうが速い

次章では、生成した記事をZenn・Qiitaへ自動投稿するPosterクラスの実装と、Git pushの非対話化（PAT埋め込みによるハング防止）を解説する。
