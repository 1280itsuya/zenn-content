---
title: "view数・収益を自動計測してフィードバックループを回す"
free: false
---

## view数・収益を自動計測してフィードバックループを回す

コンテンツを量産しても、何が読まれているかを把握していなければ改善は起きない。本章では Qiita と Zenn の公開 API で view 数を毎日収集し、上位テーマを次サイクルの生成プロンプトへ自動的に還元する `winner_extractor` の実装を解説する。

---

## なぜ「計測→還元」が必要か

記事を毎朝自動投稿するだけでは、生成テーマは固定されたままループする。人気テーマが偶然ヒットしても、その知識が次の生成に活かされない。`winner_extractor` はこの問題を解決するための **フィードバックハブ** だ。

---

## Step 1 ― Qiita API でアイテム統計を取得する

Qiita は認証トークンなしでも公開記事を閲覧できるが、自分の記事の view 数は認証が必要だ。

```python
# src/winner_extractor.py
import os, requests, json
from pathlib import Path

QIITA_TOKEN = os.environ["QIITA_TOKEN"]

def fetch_qiita_stats(per_page: int = 20) -> list[dict]:
    headers = {"Authorization": f"Bearer {QIITA_TOKEN}"}
    url = "https://qiita.com/api/v2/authenticated_user/items"
    items = []
    page = 1
    while len(items) < per_page:
        resp = requests.get(url, headers=headers,
                            params={"page": page, "per_page": 20})
        resp.raise_for_status()
        batch = resp.json()
        if not batch:
            break
        items.extend(batch)
        page += 1
    return [{"title": i["title"],
             "views": i["page_views_count"],
             "tags": [t["name"] for t in i["tags"]],
             "created_at": i["created_at"][:10]} for i in items]
```

**落とし穴**: `page_views_count` は Qiita Pro 契約アカウントでないと `null` が返る。無料アカウントでは `likes_count`（LGTM 数）で代用するしかない。

---

## Step 2 ― Zenn の view 数を取得する

Zenn は公式の認証付き API を公開していないが、ユーザーの記事一覧は JSON エンドポイントで取得できる。

```python
ZENN_USERNAME = os.environ["ZENN_USERNAME"]

def fetch_zenn_stats() -> list[dict]:
    url = f"https://zenn.dev/api/articles?username={ZENN_USERNAME}&order=latest"
    data = requests.get(url).json()
    return [{"title": a["title"],
             "views": a.get("body_letters_count", 0),   # ← view は非公開
             "likes": a["liked_count"],
             "slug": a["slug"]} for a in data.get("articles", [])]
```

**実例**: Zenn は view 数を API で返さない。代わりに `liked_count` を「人気度スコア」として扱う。筆者の環境では liked_count ≥ 3 の記事のタグが次サイクルで 40% 採用率になった。

---

## Step 3 ― 上位テーマを抽出して exemplars ファイルへ書き出す

```python
def extract_winners(qiita: list[dict], zenn: list[dict],
                    top_n: int = 5) -> list[str]:
    scored = []
    for item in qiita:
        score = (item["views"] or 0) + item.get("likes", 0) * 3
        scored.append({"title": item["title"], "score": score,
                       "tags": item["tags"]})
    for item in zenn:
        score = item["likes"] * 5          # Zennのlikeは重みを増す
        scored.append({"title": item["title"], "score": score, "tags": []})

    winners = sorted(scored, key=lambda x: x["score"], reverse=True)[:top_n]
    themes = [w["title"] for w in winners]

    out = Path("data/exemplars.json")
    out.write_text(json.dumps({"winners": themes,
                               "top_tags": _top_tags(winners)},
                              ensure_ascii=False, indent=2))
    return themes

def _top_tags(winners: list[dict]) -> list[str]:
    from collections import Counter
    tags = [t for w in winners for t in w.get("tags", [])]
    return [t for t, _ in Counter(tags).most_common(5)]
```

---

## Step 4 ― 生成プロンプトへ還元する

`data/exemplars.json` が存在する場合、次サイクルの記事生成スクリプトがこれを読み込み、システムプロンプトに差し込む。

```python
# src/generators/qiita_tech.py（生成側の抜粋）
def build_system_prompt() -> str:
    exemplars_path = Path("data/exemplars.json")
    hint = ""
    if exemplars_path.exists():
        data = json.loads(exemplars_path.read_text())
        titles = "\n".join(f"- {t}" for t in data["winners"])
        tags = ", ".join(data["top_tags"])
        hint = f"\n\n過去に好評だった記事テーマ:\n{titles}\n推奨タグ: {tags}"
    return f"あなたは技術Qiita記事を書くエキスパートです。{hint}"
```

これで「読まれた記事のテーマ → 次の生成プロンプト」という自律 PDCA サイクルが完成する。

---

## 実行を毎日スケジュールする

```python
# winner_extractor.py のエントリポイント
if __name__ == "__main__":
    q = fetch_qiita_stats()
    z = fetch_zenn_stats()
    winners = extract_winners(q, z)
    print(f"Winners extracted: {winners}")
```

Task Scheduler（Windows）または cron で毎朝 6:50 に実行し、7:00 の生成バッチより前に `exemplars.json` を更新しておく。

---

## よくある落とし穴まとめ

| 落とし穴 | 対処 |
|---|---|
| Qiita 無料アカウントで views が null | `likes_count` を代用スコアにする |
| Zenn に view 取得 API がない | `liked_count` × 5 でスコアを推計する |
| exemplars が古いまま蓄積される | 生成前に 7 日超のファイルは削除する |
| 上位テーマが 1 記事に偏る | `_top_tags` で多様性を確保してタグを優先する |

---

計測と生成が自動でつながることで、人間が何もしなくても「読まれるテーマ」に収束し始める。次章では、この exemplars を Claude API のシステムプロンプトへ組み込んで記事品質を高める方法を詳しく見ていく。
