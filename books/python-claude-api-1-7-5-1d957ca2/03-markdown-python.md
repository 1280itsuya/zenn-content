---
title: "キーワード選定からMarkdown出力までPythonで全自動化する"
free: false
---

## キーワード選定からMarkdown出力までPythonで全自動化する

アフィリエイト記事で収益を上げるには「誰も見ない良文」より「検索で拾われる記事」が先決だ。この章では、キーワードの取得から最終的なMarkdownファイル保存まで、人手を一切介さずに処理するパイプラインを構築する。

---

## 1. キーワード候補をAPIで取得する

Google Suggest（オートコンプリート）は無料で使える最速のキーワードソースだ。公式Search Console APIではなく、サジェスト用の非公式エンドポイントを叩く。

```python
import requests

def get_suggestions(seed: str) -> list[str]:
    url = "https://suggestqueries.google.com/complete/search"
    params = {"client": "firefox", "q": seed, "hl": "ja"}
    res = requests.get(url, params=params, timeout=5)
    return res.json()[1]  # ["seed ...", "seed ...", ...]

keywords = get_suggestions("Python 副業")
# → ['Python 副業 月収', 'Python 副業 初心者', 'Python 副業 案件', ...]
```

**落とし穴**: `client=chrome` を使うとJSONではなくJSONP形式で返ってくるためパースに失敗する。`firefox` を指定すること。

---

## 2. タイトルをClaude APIで生成する

取得したキーワードをClaude APIに渡し、クリック率を最大化するタイトルを生成させる。

```python
import anthropic

client = anthropic.Anthropic()

def generate_title(keyword: str) -> str:
    prompt = (
        f"キーワード「{keyword}」で検索する読者に刺さる"
        "アフィリエイト記事タイトルを1つだけ出力してください。"
        "数字・具体性・ベネフィットを入れ、40字以内にすること。"
    )
    msg = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=100,
        messages=[{"role": "user", "content": prompt}],
    )
    return msg.content[0].text.strip()

title = generate_title("Python 副業 月収")
# → "Python副業で月収5万を達成する3ステップ【初心者でも90日で実現】"
```

**ポイント**: タイトルのみ生成させる呼び出しを本文生成と分けておくと、後でA/Bテスト差し替えが楽になる。

---

## 3. 本文＋アフィリリンクを一括生成する

本文生成プロンプトには、タイトル・文字数・アフィリリンクの挿入指示を同時に渡す。

```python
AFFILIATE_LINKS = {
    "プログラミングスクール": "https://px.a8.net/svt/ejp?a8mat=XXXXX",
    "Python入門書": "https://amzn.to/XXXXX",
}

def generate_article(title: str, keyword: str) -> str:
    links_str = "\n".join(
        f"- {name}: {url}" for name, url in AFFILIATE_LINKS.items()
    )
    prompt = f"""
以下の条件でアフィリエイト記事本文をMarkdown形式で書いてください。

タイトル: {title}
メインキーワード: {keyword}
文字数: 1200〜1500字
アフィリリンク（自然な文脈で1〜2箇所に挿入）:
{links_str}

## と ### の見出し、箇条書き、まとめセクションを含めること。
"""
    msg = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2000,
        messages=[{"role": "user", "content": prompt}],
    )
    return msg.content[0].text
```

**落とし穴**: リンクをプロンプトに直接埋めるだけでは、Claudeが意図せずリンクを省略することがある。`messages.create` の後に `assert "a8.net" in body` でリンク挿入を検証し、失敗時はリトライするガードを追加すること。

---

## 4. Markdownファイルに保存する

タイトルをファイル名に使い、`output/` ディレクトリへ保存する。

```python
import re
from pathlib import Path
from datetime import date

def save_article(title: str, body: str) -> Path:
    slug = re.sub(r"[^\w\u3040-\u9fff]", "_", title)[:40]
    filename = f"{date.today()}_{slug}.md"
    path = Path("output") / filename
    path.parent.mkdir(exist_ok=True)

    full_content = f"# {title}\n\n{body}"
    path.write_text(full_content, encoding="utf-8")
    return path
```

---

## 5. パイプラインとして一気通貫に動かす

```python
def run_pipeline(seed: str):
    keywords = get_suggestions(seed)[:3]  # 上位3件に絞る
    for kw in keywords:
        title = generate_title(kw)
        body  = generate_article(title, kw)
        path  = save_article(title, body)
        print(f"保存完了: {path}")

run_pipeline("Python 副業")
```

実行すると `output/2026-06-15_Python副業で月収5万を達成する3ステップ.md` のようなファイルが3本生成される。1キーワードあたりのAPI呼び出しは2回（タイトル＋本文）で、Claude Sonnetの料金換算では1記事あたり約2〜4円。生成時間は通信込みで7分前後に収まる。

**最終的な落とし穴**: `output/` をそのままGitHubに上げるとアフィリリンクがクローラーに拾われてBANされるケースがある。`.gitignore` に `output/` を追加しておくこと。
