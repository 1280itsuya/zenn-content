---
title: "記事生成エージェントの実装――テーマ選定から本文出力まで"
free: false
---

## 記事生成エージェントの実装――テーマ選定から本文出力まで

### 何を作るか

このエージェントは「勝ちタグ」を受け取り、タイトル生成 → 本文執筆 → タグ付けを1スクリプトで完走する。入力は `["Python", "Claude", "個人開発"]` のようなリスト1本。出力はそのまま Zenn や Qiita に投稿できる Markdown ファイルだ。

---

### 全体構造

```
generate_article.py
  ├── select_theme()   # タグ×トレンドからテーマを決定
  ├── draft_title()    # テーマからタイトル3案を生成し最良を選択
  ├── write_body()     # タイトルを受けて本文を出力
  └── assign_tags()    # 本文を読んでタグを付与
```

各ステップを別プロンプトに分割するのがポイントだ。1回の呼び出しで全部やらせると、タイトルと本文が乖離しやすい。

---

### 実装

```python
import anthropic
import json
from pathlib import Path

client = anthropic.Anthropic()  # ANTHROPIC_API_KEY を .env から読む

WINNING_TAGS = ["Python", "Claude", "個人開発"]

def select_theme(tags: list[str]) -> str:
    resp = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=256,
        messages=[{
            "role": "user",
            "content": (
                f"タグ: {tags}\n"
                "このタグで検索する開発者が今最も知りたいテーマを1行で答えよ。"
                "例: 『Claude APIのストリーミングをFastAPIで受け取る実装』"
            )
        }]
    )
    return resp.content[0].text.strip()

def draft_title(theme: str) -> str:
    resp = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=256,
        messages=[{
            "role": "user",
            "content": (
                f"テーマ: {theme}\n"
                "Zenn向けタイトルを3案JSON配列で出力せよ。"
                "例: [\"...\", \"...\", \"...\"]"
            )
        }]
    )
    titles = json.loads(resp.content[0].text.strip())
    # 最初の案を採用（後続でA/Bテストに拡張可能）
    return titles[0]

def write_body(title: str) -> str:
    resp = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        system=(
            "あなたは現役の個人開発者です。"
            "タイトルに厳密に沿い、コード例・落とし穴・実測値を含む"
            "Markdown技術記事を1500〜2000字で書いてください。"
        ),
        messages=[{"role": "user", "content": f"タイトル: {title}"}]
    )
    return resp.content[0].text.strip()

def assign_tags(body: str, candidates: list[str]) -> list[str]:
    resp = client.messages.create(
        model="claude-haiku-4-5-20251001",  # タグ付けは軽量モデルで十分
        max_tokens=128,
        messages=[{
            "role": "user",
            "content": (
                f"候補タグ: {candidates}\n"
                f"本文(先頭300字): {body[:300]}\n"
                "最適なタグを5個以内でJSON配列として返せ。"
            )
        }]
    )
    return json.loads(resp.content[0].text.strip())

def run(tags: list[str] = WINNING_TAGS) -> Path:
    theme  = select_theme(tags)
    title  = draft_title(theme)
    body   = write_body(title)
    chosen = assign_tags(body, tags + ["API", "自動化", "副業"])

    # タイトルと本文が乖離していないか検証
    if title.split("。")[0][:10] not in body[:500]:
        raise ValueError(f"本文がタイトルから逸れています: {title}")

    out = Path(f"articles/{title[:40]}.md")
    out.parent.mkdir(exist_ok=True)
    out.write_text(f"---\ntitle: {title}\ntags: {chosen}\n---\n\n{body}")
    print(f"生成完了: {out}  ({len(body)}字)")
    return out

if __name__ == "__main__":
    run()
```

---

### 落とし穴と対策

**タイトルと本文のドリフト**  
`write_body` にタイトルだけを渡すと、モデルが途中で話題を変えることがある。`system` プロンプトに「タイトルに厳密に沿え」と明記し、生成後にタイトルの冒頭10文字が本文前半に含まれるかをチェックする（上記コードの `raise ValueError` 部分）。不一致なら投稿をせず再生成を促す。

**タグ付けコストの節約**  
タグ判定は意味理解よりも語彙照合に近い作業なので、`claude-haiku-4-5` で十分だ。`claude-sonnet-4-6` を使い続けると1記事あたりのAPIコストが約3倍になる。

**JSON パースエラー**  
モデルが JSON 以外のテキストを前置きすることがある。正規表現で `\[.*\]` を抜き出す防御処理を加えておくと安定する。

```python
import re

def safe_parse(text: str) -> list:
    m = re.search(r'\[.*?\]', text, re.DOTALL)
    return json.loads(m.group()) if m else []
```

---

### 実行例

```
$ python generate_article.py
生成完了: articles/FastAPIでClaudeのストリーミングを受け取る最速実装.md  (1843字)
```

手元環境では1記事あたり約7分（生成4分 + API待機3分）で完走する。タグを差し替えるだけで別ニッチの記事に切り替わるため、日次バッチで複数タグを順に流す拡張も容易だ。
