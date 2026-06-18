---
title: "第6章：収益化への配線——アフィリリンク埋め込み・Gumroad連携・view数から逆算する改善ループ"
free: false
---

## 第6章：収益化への配線——アフィリリンク埋め込み・Gumroad連携・view数から逆算する改善ループ

記事を量産しても、収益への配線がなければ空振りになる。この章では「生成→挿入→出品→計測→再生成」のループを一本のパイプラインとして閉じる。

---

## アフィリリンクの自動挿入

アフィリリンクをプロンプトに書かせると、モデルは文脈を無視した不自然な挿入をしやすい。推奨パターンは**後処理で差し込む**方式だ。

```python
AFFILIATE_MAP = {
    "Python入門": "https://amzn.to/XXXXX",
    "Claude API": "https://a8.net/XXXXX",
}

def inject_links(body: str) -> str:
    for keyword, url in AFFILIATE_MAP.items():
        # 最初の1箇所だけリンク化（スパム回避）
        body = body.replace(keyword, f"[{keyword}]({url})", 1)
    return body
```

**落とし穴**：同じキーワードが複数回出てくると全置換になりスパム判定される。`replace(..., 1)` で初回のみに限定すること。また、A8やアマゾンアソシエイトは審査基準として「コンテンツが先、リンクが従」を求める。リンク密度は本文1,000字あたり2〜3個が上限の目安。

---

## Gumroadへの自動出品

Claude APIで生成したPDF教材やコードキットをGumroadに自動出品する。Gumroadは `presigned_url` を取得してからファイルをアップロードするフローになっている。

```python
import requests, os

def publish_to_gumroad(name: str, pdf_path: str, price_cents: int = 98000):
    token = os.environ["GUMROAD_TOKEN"]
    # 1. 商品作成
    res = requests.post(
        "https://api.gumroad.com/v2/products",
        data={"name": name, "price": price_cents},
        headers={"Authorization": f"Bearer {token}"},
    )
    product_id = res.json()["product"]["id"]

    # 2. presigned URL取得 → ファイルPUT
    presign = requests.get(
        f"https://api.gumroad.com/v2/products/{product_id}/product_files/presign",
        headers={"Authorization": f"Bearer {token}"},
    ).json()
    with open(pdf_path, "rb") as f:
        requests.put(presign["url"], data=f)
    return product_id
```

**実例**：`jsconfig-diag-kit`（JS/TS設定エラーAI診断キット、¥980）はこのコードで毎週自動出品している。**ブロッカーは支払い方法の接続**——コードの問題ではなくGumroadのアカウント設定が未完だと「下書き保存」で止まる。必ず事前に確認しておくこと。

---

## Zennビュー数から逆算する改善ループ

「何が読まれているか」を知らずに生成テーマを決めるのは闇雲な量産になる。Zennの統計APIでビュー数を取得し、上位テーマを次サイクルの生成指示に注入する。

```python
import requests, os

def fetch_top_articles(username: str, top_n: int = 5) -> list[dict]:
    url = f"https://zenn.dev/api/articles?username={username}&order=latest"
    articles = requests.get(url).json()["articles"]
    return sorted(articles, key=lambda a: a["liked_count"], reverse=True)[:top_n]

def build_strategy_prompt(top_articles: list[dict]) -> str:
    titles = "\n".join(f"- {a['title']} ({a['liked_count']} likes)" for a in top_articles)
    return f"以下は過去の人気記事です。同ジャンルで深掘りした新テーマを5つ提案してください:\n{titles}"
```

これを毎日7時のオーケストレーターに組み込むと、ビュー数が高い記事のジャンルが翌日の生成テーマに自動反映される。**注意点**：Zennはいいね数は公開APIで取れるが詳細ビュー数はダッシュボードのみ。いいね数を代理指標として使い、週次で手動確認する運用が現実的だ。

---

## ループをつなぐ

```
生成（Claude API）
  ↓
後処理（アフィリリンク挿入）
  ↓
投稿（Zenn / Qiita / はてな）
  ↓
計測（likes / view取得）
  ↓
winner_extractor → 次回の生成プロンプトへ注入
```

このループが1週間回ると、勝ちテーマが自然に濃縮されていく。最初の1〜2週は全ジャンルを試す「探索フェーズ」、3週目以降は上位ジャンルに絞る「増幅フェーズ」に移行するのが効率的だ。
