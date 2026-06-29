---
title: "第3章　Claude APIでコンテンツ生成ロジックを実装するコアパイプライン"
free: false
---

## 第3章　Claude APIでコンテンツ生成ロジックを実装するコアパイプライン

本章では、テーマ選定からMarkdownファイル出力まで、コンテンツ自動生成の中核となるコードを順番に組み立てる。

---

## Step 1: テーマ選定ロジック

最初の落とし穴は「テーマを固定文字列にしてしまう」ことだ。これでは1週間で枯渇する。代わりに、日付ベースのシードでニッチを循環させる。

```python
# theme_selector.py
import datetime

NICHES = [
    "Python副業", "Claude API活用", "AI自動化ツール",
    "個人開発収益化", "Zenn記事量産", "ローコスト起業"
]

def pick_theme(date: datetime.date = None) -> str:
    if date is None:
        date = datetime.date.today()
    idx = date.toordinal() % len(NICHES)
    return NICHES[idx]
```

`toordinal()` で日付を整数変換することで、同じ日は必ず同じテーマ、翌日は自動で次のテーマに進む。ランダム値を使わないのは**再現性**を確保するためだ。デバッグ時に引数で日付を渡せばいつでも同じ結果を得られる。

---

## Step 2: プロンプト設計

Claude APIへ渡すプロンプトは「役割・制約・出力形式」の3要素で構成する。特に**出力形式の明示**を省くと、記事構造がランダムになり後続処理が壊れる。

```python
# prompt_builder.py
def build_prompt(theme: str) -> str:
    return f"""あなたは技術ブログの専門ライターです。

テーマ: {theme}

以下の形式でMarkdown記事を書いてください。

# [具体的なタイトル]

## はじめに
（読者の悩みを1段落で提示）

## 本題
（コード例を含む具体的な解説、500字以上）

## まとめ
（行動を促す締め）

制約:
- タイトルは「{theme}」の内容と完全一致させること
- コード例は必ず動作する実コードにすること
- 宣伝文句や誇大表現を使わないこと
"""
```

「タイトルは内容と完全一致させること」という制約は必須だ。これを入れないと、タイトルは「Python副業」なのに本文は別テーマに漂流する**ドリフト**が頻発する。

---

## Step 3: Claude API呼び出し

```python
# generator.py
import anthropic
from pathlib import Path

def generate_article(theme: str) -> str:
    client = anthropic.Anthropic()  # ANTHROPIC_API_KEYを環境変数から自動読込
    prompt = build_prompt(theme)

    message = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        messages=[{"role": "user", "content": prompt}]
    )
    return message.content[0].text
```

**落とし穴**: `message.content[0].text` を直接使うこと。`message.content` はリストであり、インデックスを省くと文字列ではなくオブジェクトが返って後続処理でクラッシュする。

レートリミット対策として、1日複数記事を生成する場合は呼び出し間に `time.sleep(3)` を挟む。無料枠ではなくMax定額プランであっても、並列リクエストの同時多発は避けること。

---

## Step 4: Markdown出力

```python
# output.py
import datetime
from pathlib import Path

def save_article(content: str, theme: str) -> Path:
    today = datetime.date.today().isoformat()  # 例: "2026-06-29"
    # テーマをファイル名に使う際は記号を除去
    safe_theme = theme.replace("/", "_").replace(" ", "_")
    filename = f"{today}_{safe_theme}.md"

    output_dir = Path("output/articles")
    output_dir.mkdir(parents=True, exist_ok=True)

    filepath = output_dir / filename
    filepath.write_text(content, encoding="utf-8")
    return filepath
```

`mkdir(parents=True, exist_ok=True)` を毎回呼ぶことで、ディレクトリ存在チェックを省略できる。初回実行でも2回目以降でもエラーにならない。

---

## パイプラインの統合

```python
# main.py
import datetime
from theme_selector import pick_theme
from generator import generate_article
from output import save_article

def run():
    theme = pick_theme()
    print(f"[テーマ] {theme}")

    content = generate_article(theme)

    # 最低限の品質ゲート: タイトル行にテーマが含まれているか検証
    first_line = content.split("\n")[0]
    if theme not in first_line:
        raise ValueError(f"タイトルドリフト検出: '{first_line}' にテーマが含まれていない")

    path = save_article(content, theme)
    print(f"[保存] {path}")

if __name__ == "__main__":
    run()
```

品質ゲートの `ValueError` は意図的にクラッシュさせている。エラーを握りつぶして不正記事を保存するより、**止めて気づかせる**方が運用コストが低い。Task Schedulerやcronで自動実行する場合、ログに例外が残るため翌朝確認できる。

---

## この章のポイント

| 設計判断 | 理由 |
|---|---|
| 日付シードでテーマ循環 | ランダム避け・再現性確保 |
| プロンプトに出力形式を明示 | 後続処理の壊れ防止 |
| タイトルドリフト検証 | 品質維持・無駄なAPI費用削減 |
| `exist_ok=True` で冪等性 | 初回・再実行どちらでも安全 |

次章では、この `output/articles/` に溜まったMarkdownファイルをZennやQiitaへ自動投稿するポスターモジュールを実装する。
