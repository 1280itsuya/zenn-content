---
title: "収益化と改善ループ──view上位テーマを次の生成に還元するウィナー増幅設計"
free: false
---

## 収益化と改善ループ──view上位テーマを次の生成に還元するウィナー増幅設計

AI生成パイプラインは「量産できる」だけでは収益にならない。view0の記事を毎日10本出しても、誰にも読まれなければアフィリリンクはクリックされない。この章では、実計測viewからシグナルを拾い、次の生成サイクルへ自動反映する**ウィナー増幅ループ**と、CTA配線による収益化の実装を解説する。

---

## winner_extractor：viewランキングから勝ちタグを抽出する

ZennやQiitaのAPIを叩いて、直近30日の自記事view数を取得し、上位タグ・キーワードを`exemplars.json`に書き出すスクリプトが`winner_extractor`だ。

```python
# src/winner_extractor.py
import os, json, requests
from dotenv import load_dotenv

load_dotenv()
ZENN_USERNAME = os.environ["ZENN_USERNAME"]

def fetch_zenn_articles() -> list[dict]:
    url = f"https://zenn.dev/api/articles?username={ZENN_USERNAME}&order=liked_count"
    resp = requests.get(url, timeout=10)
    resp.raise_for_status()
    return resp.json().get("articles", [])

def extract_winners(articles: list[dict], top_n: int = 5) -> dict:
    sorted_articles = sorted(articles, key=lambda a: a.get("liked_count", 0), reverse=True)
    top = sorted_articles[:top_n]
    
    tag_scores: dict[str, int] = {}
    for article in top:
        for topic in article.get("topics", []):
            slug = topic["slug"]
            tag_scores[slug] = tag_scores.get(slug, 0) + article["liked_count"]
    
    return {
        "top_tags": sorted(tag_scores, key=tag_scores.get, reverse=True)[:10],
        "exemplar_titles": [a["title"] for a in top],
        "updated_at": __import__("datetime").date.today().isoformat(),
    }

if __name__ == "__main__":
    articles = fetch_zenn_articles()
    winners = extract_winners(articles)
    out_path = "data/exemplars.json"
    with open(out_path, "w", encoding="utf-8") as f:
        json.dump(winners, f, ensure_ascii=False, indent=2)
    print(f"Saved winners → {out_path}")
    print("Top tags:", winners["top_tags"])
```

> **落とし穴①**：Zenn APIはページネーションが必要。`page=1`のみ取得すると最新20件しか見えず、長期実績記事を取り逃す。`while`ループで`next_page`が`null`になるまで全ページ取得すること。

---

## 生成プロンプトへの自動反映

`exemplars.json`をstrategistエージェントのシステムプロンプトに差し込む。

```python
# src/agents/strategist.py の抜粋
import json, pathlib

def build_system_prompt() -> str:
    exemplars_path = pathlib.Path("data/exemplars.json")
    if exemplars_path.exists():
        exemplars = json.loads(exemplars_path.read_text(encoding="utf-8"))
        tag_hint = "、".join(exemplars["top_tags"][:5])
        title_hint = "\n".join(f"- {t}" for t in exemplars["exemplar_titles"])
        winner_block = f"""
## 過去view上位から学ぶ
読まれやすいタグ（優先使用）: {tag_hint}
参考タイトル構造:
{title_hint}
"""
    else:
        winner_block = ""

    return f"""あなたはZenn/Qiita向けAI技術記事のトピック立案者です。
{winner_block}
上記の傾向を踏まえ、今日の生成テーマを1つ提案してください。
テーマはタイトル・タグ・200字概要の形式で返してください。"""
```

これにより、毎朝7時の自動実行で「先週よく読まれたタグ」が今日の記事テーマに反映される。**人手ゼロでデータ駆動のA/Bサイクルが回る**。

> **落とし穴②**：`liked_count`はLGTM数であり純粋viewではない。Zennの`page_views_count`フィールドは自分の記事管理画面APIにのみ存在する。Qiitaは`/api/v2/authenticated_user/items`でview数取得可能。両方取ってスコアを正規化するとシグナルが安定する。

---

## 収益CTA──アフィリ・Booth商品への送客を配線する

記事末尾に毎回手動でリンクを貼るのは持続しない。生成テンプレートにCTAブロックを埋め込む。

```python
# src/agents/writer.py の抜粋
CTA_BLOCK = """
---
## 関連ツール・教材

- **[Claude×MCP導入キット（Booth ¥2,480）](https://booth.pm/ja/items/XXXXXX)**  
  本記事で紹介した設定をそのまま使えるスターターキット。`.claude/settings.json`サンプル付き。

- **[llms.txt自動生成ツール（Booth ¥1,980）](https://booth.pm/ja/items/YYYYYY)**  
  サイトのAI検索最適化を一発設定。

- **AI業務自動化の無料相談** → [お問い合わせ](/service/)
"""

def generate_article(topic: dict, client) -> str:
    # ... 本文生成 ...
    body = call_claude(topic, client)
    return body + "\n" + CTA_BLOCK
```

> **落とし穴③**：Zennはmarkdown中の外部リンクに`rel="nofollow"`を付与する。SEOへの直接効果は薄いが、**クリック経由の購入**は計測できる。BoothのURLパラメータ（`?ref=zenn`）を付けることでどの記事からの流入かをBoothの販売履歴と突き合わせられる。

---

## 自動化フロー全体の接続

```
07:00  winner_extractor.py 実行  →  data/exemplars.json 更新
07:05  orchestrator 起動
       └─ strategist（exemplars参照）→ テーマ決定
       └─ writer（CTA付き本文生成）
       └─ zenn_poster / qiita_poster → 公開
07:30  Zenn/Qiita にview付き記事が蓄積
翌週   winner_extractor が再実行 → ループ継続
```

月1回、`exemplars.json`の`top_tags`を手で確認して「意図せず偏った方向に収束していないか」を見る。同じタグに集中しすぎると新規流入が止まるため、上位5タグのうち1〜2枠は**探索枠**としてランダムに新タグを混ぜる設計にすると長期的に安定する。
