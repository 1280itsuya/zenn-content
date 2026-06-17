---
title: "第3章：Pythonでネタ収集→構成→本文生成を一気通貫するパイプライン実装"
free: false
---

## 第3章：Pythonでネタ収集→構成→本文生成を一気通貫するパイプライン実装

アフィリ記事の量産で最初に詰まるのが「ネタ切れ」だ。手動で調べていては1日3本が限界になる。この章では、RSSフィードとキーワードAPIでネタを自動収集し、Claude APIで見出し構成と本文を連続生成するスクリプトを一気に組む。

---

## ネタ収集：RSSとキーワードAPIを組み合わせる

Google Newsのカテゴリ別RSSから最新トピックを引き、それをSuggestAPIで展開してロングテールキーワードを得る流れが基本だ。

```python
import feedparser
import requests

RSS_URLS = [
    "https://news.google.com/rss/search?q=副業&hl=ja&gl=JP&ceid=JP:ja",
    "https://news.google.com/rss/search?q=節約&hl=ja&gl=JP&ceid=JP:ja",
]

def fetch_topics(max_per_feed: int = 5) -> list[str]:
    topics = []
    for url in RSS_URLS:
        feed = feedparser.parse(url)
        for entry in feed.entries[:max_per_feed]:
            topics.append(entry.title)
    return topics

def expand_keywords(seed: str) -> list[str]:
    url = "https://suggestqueries.google.com/complete/search"
    params = {"client": "firefox", "hl": "ja", "q": seed}
    res = requests.get(url, params=params, timeout=5)
    return res.json()[1][:5]  # サジェスト上位5件
```

**落とし穴①** — `feedparser`はエンコードエラーで無言に止まることがある。`entry.get("title", "")`でデフォルト値を必ず入れること。

---

## 構成生成：Claude APIで見出しツリーを作る

収集したキーワードをそのままプロンプトに渡すのではなく、「読者の悩み→解決策→比較→行動」の型をシステムプロンプトに焼き込む。これだけで構成の揺れが激減する。

```python
import anthropic

client = anthropic.Anthropic()

def generate_outline(keyword: str) -> str:
    system = (
        "あなたはSEOアフィリ記事の構成専門家です。"
        "必ず「課題提起→原因分析→解決策3選→比較表→まとめ」の5ブロックで"
        "H2見出しを5本出力してください。余計な説明は不要。"
    )
    msg = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=512,
        system=system,
        messages=[{"role": "user", "content": f"キーワード：{keyword}"}],
    )
    return msg.content[0].text
```

**落とし穴②** — 構成生成にOpusを使うと1本あたりのコストが跳ね上がる。構成はHaikuで十分。本文だけSonnetに切り替えると品質とコストのバランスが取れる。

---

## 本文生成：見出しを1本ずつストリーミングで書かせる

見出しを一括で渡すと後半ほど内容が薄くなる。H2ごとにAPIを呼び、前のブロックの末尾200文字を`context`として渡すと文章の連続性が保たれる。

```python
def generate_section(h2: str, context: str = "") -> str:
    prompt = (
        f"前のセクションの末尾：\n{context}\n\n"
        f"次のH2「{h2}」について400字で本文を書いてください。"
        "アフィリリンクの挿入箇所は[[AFFILIATE]]と記入。"
    )
    msg = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}],
    )
    return msg.content[0].text

def build_article(keyword: str) -> str:
    outline_raw = generate_outline(keyword)
    headings = [l for l in outline_raw.splitlines() if l.startswith("##")]
    
    sections, context = [], ""
    for h2 in headings:
        body = generate_section(h2, context[-200:])
        sections.append(f"{h2}\n\n{body}")
        context = body  # 次のセクションへ引き継ぐ
    
    return "\n\n".join(sections)
```

---

## 一気通貫で動かす

```python
if __name__ == "__main__":
    topics = fetch_topics()
    for topic in topics[:3]:  # 1回あたり3本
        kws = expand_keywords(topic)
        keyword = kws[0] if kws else topic
        article = build_article(keyword)
        
        # ファイル名は日付+キーワードのslug
        slug = keyword.replace(" ", "_")[:30]
        with open(f"output/{slug}.md", "w", encoding="utf-8") as f:
            f.write(f"# {keyword}\n\n{article}")
        print(f"生成完了: {slug}")
```

**落とし穴③** — `output/`ディレクトリが存在しないと`FileNotFoundError`で全体が止まる。スクリプト冒頭で`os.makedirs("output", exist_ok=True)`を入れておく。

---

## 実行コストの目安

| モデル | 用途 | 1記事あたり |
|---|---|---|
| Haiku 4.5 | 構成生成 | 約¥0.3 |
| Sonnet 4.6 | 本文5セクション | 約¥8 |
| **合計** | | **約¥8.3/本** |

月100本なら約830円。RSS→サジェスト→構成→本文のチェーンが1スクリプトで完結し、`cron`に乗せれば毎朝7時に自動で記事ファイルが溜まっていく状態が作れる。次章ではこのファイルをZennとはてなブログへ自動投稿する配管を組む。
