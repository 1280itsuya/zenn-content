---
title: "コア実装：キーワード入力→記事生成の自動化スクリプト"
free: false
---

## コア実装：キーワード入力→記事生成の自動化スクリプト

### 完成形のイメージ

```
$ python pipeline.py --keyword "Python 初心者 おすすめ 本"
→ outline.md  （アウトライン）
→ article.md  （本文＋アフィリリンク挿入済み）
```

1ファイルで「アウトライン生成→本文展開→リンク挿入→Markdown出力」を一気通貫に処理する。以下にその全コードを示す。

---

### ステップ①：アウトラインをLLMに生成させる

```python
import anthropic, os, re, json

client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

def generate_outline(keyword: str) -> list[str]:
    prompt = f"""
あなたはSEOライターです。
キーワード「{keyword}」で検索ユーザーが求める情報を網羅した
アフィリエイト記事のH2見出しを5つ生成してください。
JSON配列で返してください。例: ["見出し1","見出し2",...]
"""
    msg = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=512,
        messages=[{"role": "user", "content": prompt}],
    )
    raw = msg.content[0].text
    # コードブロック除去してパース
    raw = re.sub(r"```.*?```", lambda m: m.group().split("\n", 1)[1].rsplit("\n", 1)[0], raw, flags=re.S)
    return json.loads(raw)
```

**落とし穴①** ：LLMはコードブロック（` ```json ` …）で包んで返すことが多い。`re.sub`で除去してからパースしないと`json.JSONDecodeError`が出る。

---

### ステップ②：見出しごとに本文を展開する

```python
def expand_section(keyword: str, heading: str) -> str:
    prompt = f"""
記事テーマ:「{keyword}」
見出し:「{heading}」

この見出しに対応する本文を300〜400文字で書いてください。
・具体例や数字を入れること
・読者目線で「なぜ重要か」を先に書くこと
・Markdownは使わず、プレーンテキストで返すこと
"""
    msg = client.messages.create(
        model="claude-haiku-4-5-20251001",  # 量産フェーズはコスト優先
        max_tokens=600,
        messages=[{"role": "user", "content": prompt}],
    )
    return msg.content[0].text.strip()
```

**モデル使い分けの原則**：アウトライン（判断が必要）はSonnet、本文展開（量産）はHaikuにするとAPI費用を約70%削減できる。

---

### ステップ③：アフィリリンクをルールベースで挿入する

```python
AFFILIATE_MAP = {
    "Python": "https://amzn.to/XXXXX",
    "書籍": "https://amzn.to/YYYYY",
    "おすすめ": "https://amzn.to/ZZZZZ",
}

def inject_affiliate_links(text: str) -> str:
    inserted = set()
    for kw, url in AFFILIATE_MAP.items():
        if kw in text and kw not in inserted:
            # 最初の1回だけリンク化（過剰挿入を防ぐ）
            text = text.replace(kw, f"[{kw}]({url})", 1)
            inserted.add(kw)
    return text
```

**落とし穴②** ：`replace`を無制限に使うと同一記事内に同じリンクが10回出現しスパム判定される。`inserted`セットで「1キーワード1挿入」を強制すること。

---

### ステップ④：全体を組み上げてMarkdown出力

```python
import argparse, pathlib

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--keyword", required=True)
    parser.add_argument("--out", default="article.md")
    args = parser.parse_args()

    print(f"[1/3] アウトライン生成中...")
    headings = generate_outline(args.keyword)

    print(f"[2/3] 本文展開中（{len(headings)}セクション）...")
    sections = []
    for h in headings:
        body = expand_section(args.keyword, h)
        body_with_links = inject_affiliate_links(body)
        sections.append(f"## {h}\n\n{body_with_links}")

    article = f"# {args.keyword}\n\n" + "\n\n".join(sections)

    pathlib.Path(args.out).write_text(article, encoding="utf-8")
    print(f"[3/3] 出力完了 → {args.out}")

if __name__ == "__main__":
    main()
```

---

### 実行例と出力確認

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
python pipeline.py --keyword "Python 初心者 おすすめ 本"
```

生成された`article.md`の冒頭は以下のようになる：

```markdown
# Python 初心者 おすすめ 本

## 独学で挫折しない選び方の3つのポイント

[Python](https://amzn.to/XXXXX)を学ぶ初心者が最初につまずくのは...
```

**落とし穴③** ：キーワードに記号（`+`や`/`）が含まれるとファイル名にそのまま使えない。`args.keyword`をファイル名に流用する場合は`re.sub(r'[^\w]', '_', kw)`でサニタイズすること。

---

### まとめ：このスクリプトの拡張ポイント

| 拡張項目 | 実装のヒント |
|---|---|
| キーワードをCSV一括処理 | `csv.reader`でループを回す |
| 生成済みをスキップ | `Path(out).exists()`で分岐 |
| アフィリマップを外部化 | `affiliate.json`をロード |
| 品質チェック | 文字数が200字未満なら再生成 |

次章ではこのスクリプトをTask Schedulerで毎朝自動起動し、生成した記事をZennやはてなブログへAPIで自動投稿する仕組みを実装する。
