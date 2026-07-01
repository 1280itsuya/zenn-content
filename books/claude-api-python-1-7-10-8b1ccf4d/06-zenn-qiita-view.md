---
title: "収益計測ループの作り方——Zenn/Qiitaのviewを自動収集して勝ちテーマを次回生成に還元する"
free: false
---

## 収益計測ループの作り方——Zenn/Qiitaのviewを自動収集して勝ちテーマを次回生成に還元する

記事を量産しても「何が刺さったか」を把握しないと、空振りを繰り返すだけだ。このループを閉じる仕組みが **winner_extractor** ——view上位記事のタグとタイトルパターンを自動抽出し、次サイクルの生成プロンプトへ注入するフィードバック機構だ。

---

## Zenn/QiitaのAPIでview数を取得する

Zennは公式APIを持たないため、スクレイピングで対応する。Qiitaは認証付きAPIが使える。

```python
# src/winner_extractor.py
import os, requests, json
from datetime import date

QIITA_TOKEN = os.environ["QIITA_TOKEN"]

def fetch_qiita_views(limit=20):
    """自分の投稿をview数降順で取得"""
    headers = {"Authorization": f"Bearer {QIITA_TOKEN}"}
    r = requests.get(
        "https://qiita.com/api/v2/authenticated_user/items",
        headers=headers,
        params={"per_page": limit}
    )
    r.raise_for_status()
    items = r.json()
    return sorted(items, key=lambda x: x["page_views_count"], reverse=True)
```

**落とし穴①：Qiita APIの`page_views_count`は自分の記事にしか返らない。**他人の記事を取得しても`null`になる。必ず `authenticated_user/items` を使うこと。

---

## 勝ちパターンを抽出する

view上位の記事からタグ頻度とタイトルの頭出しパターンを集計する。

```python
from collections import Counter

def extract_winners(items, top_n=5):
    winners = items[:top_n]
    tag_counter = Counter()
    title_prefixes = []

    for item in winners:
        for tag in item["tags"]:
            tag_counter[tag["name"]] += 1
        # 「Pythonで〇〇する方法」→「Pythonで」を抽出
        title_prefixes.append(item["title"][:10])

    return {
        "top_tags": [t for t, _ in tag_counter.most_common(5)],
        "title_patterns": title_prefixes,
        "as_of": str(date.today())
    }
```

**落とし穴②：直近1週間の記事だけを見ると、まだviewが育っていない新記事が「負け」判定される。**投稿から7日以上経った記事のみを対象にするフィルタを入れる。

```python
from datetime import datetime, timezone, timedelta

def filter_matured(items, days=7):
    cutoff = datetime.now(timezone.utc) - timedelta(days=days)
    return [
        i for i in items
        if datetime.fromisoformat(i["created_at"].replace("Z", "+00:00")) < cutoff
    ]
```

---

## 抽出結果を次サイクルのプロンプトに注入する

`data/winners.json` に保存し、記事生成エージェントがロードする。

```python
import json, pathlib

def save_winners(winners, path="data/winners.json"):
    pathlib.Path(path).parent.mkdir(exist_ok=True)
    with open(path, "w", encoding="utf-8") as f:
        json.dump(winners, f, ensure_ascii=False, indent=2)
```

生成側では、このJSONをプロンプトへ埋め込む：

```python
def build_prompt_with_winners(base_prompt: str, winners_path="data/winners.json") -> str:
    try:
        with open(winners_path, encoding="utf-8") as f:
            w = json.load(f)
        hint = (
            f"過去にview数上位だったタグ: {', '.join(w['top_tags'])}\n"
            f"タイトルの傾向: {'; '.join(w['title_patterns'])}\n"
            "これらを参考にしつつ、テーマは以下の指示に厳密に従え。\n\n"
        )
        return hint + base_prompt
    except FileNotFoundError:
        return base_prompt  # 初回はファイル未存在でもスキップして通過
```

**落とし穴③：winner情報を注入しすぎると、タイトルと本文がズレた「タグだけ最適化」記事が量産される。**プロンプトの末尾に必ず `テーマ: {topic}` を固定し、winnerはあくまで「傾向のヒント」として渡す。本文の冒頭でテーマ一致を確認するバリデーションも入れると安全だ。

---

## 毎朝の定期実行に組み込む

```python
# orchestrator.pyの朝タスクに追加
def morning_task():
    items   = fetch_qiita_views()
    matured = filter_matured(items)
    winners = extract_winners(matured)
    save_winners(winners)
    # 以降の生成ループはwinnersを参照して動く
```

これで「公開 → view計測 → 勝ちパターン抽出 → 次の生成へ還元」という自己強化ループが完成する。view数ゼロの記事が続くなら、タグ不一致か公開スキップのバグを疑うこと——`winners.json`の`as_of`を見て、データが古くなっていないか毎週確認するのが運用の基本だ。
