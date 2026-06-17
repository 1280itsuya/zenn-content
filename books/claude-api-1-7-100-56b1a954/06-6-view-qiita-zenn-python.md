---
title: "第6章：view数を増やすQiita/Zenn投稿最適化とPythonによる効果測定ループ"
free: false
---

## view数を増やすQiita/Zenn投稿最適化とPythonによる効果測定ループ

記事を量産しても読まれなければ意味がない。Qiita・Zennのアルゴリズムはタイトル・タグ・投稿時刻の3変数に強く反応する。この章ではその3変数をABテストし、view数をPythonで自動集計して次回生成に還元する改善サイクルを実装する。

---

## タイトルパターンのABテスト設計

Qiita・Zennどちらも「検索流入」と「タイムライン流入」の2経路がある。タイトルはその両方に効く。

**効果的なパターン3類**

| パターン | 例 | 強み |
|---|---|---|
| 疑問形 | `PythonでQiitaのview数を自動取得する方法` | 検索意図に直撃 |
| 数字+具体 | `Claude API×Python：1記事7分で量産する5ステップ` | CTR高い |
| エラー文そのまま | `TypeError: 'NoneType' object is not subscriptable の原因と直し方` | ロングテールSEO |

生成時にタイトル候補を複数出力し、デプロイ前にどれを使ったか記録する。

```python
# title_ab.py
import random, json, datetime

PATTERNS = [
    "疑問形",
    "数字+具体",
    "エラー文",
]

def pick_title(candidates: list[str]) -> dict:
    chosen = random.choice(candidates)
    pattern = classify_pattern(chosen)  # 簡易分類
    return {
        "title": chosen,
        "pattern": pattern,
        "posted_at": datetime.datetime.now().isoformat(),
    }

def classify_pattern(title: str) -> str:
    if title.endswith("？") or "方法" in title or "とは" in title:
        return "疑問形"
    if any(c.isdigit() for c in title):
        return "数字+具体"
    if "Error" in title or "Exception" in title:
        return "エラー文"
    return "その他"
```

---

## タグ選定の落とし穴

**最大の失敗パターン**: LLMが自動生成したタグはQiita公式タグと一致しないことが多い。`machine-learning` ではなく `機械学習` を使わないとフォロワーに届かない。

対策として、使用前にQiita APIでタグの存在を確認する。

```python
# tag_validator.py
import requests, os

QIITA_TOKEN = os.environ["QIITA_TOKEN"]

def validate_tags(candidates: list[str], top_n: int = 5) -> list[str]:
    """候補タグをQiita APIで検索し、実在するものだけ返す"""
    valid = []
    for tag in candidates:
        r = requests.get(
            f"https://qiita.com/api/v2/tags/{tag}",
            headers={"Authorization": f"Bearer {QIITA_TOKEN}"},
        )
        if r.status_code == 200:
            data = r.json()
            valid.append((tag, data["items_count"]))
    # フォロワー数降順で上位N個
    valid.sort(key=lambda x: x[1], reverse=True)
    return [t for t, _ in valid[:top_n]]
```

Zennはタグ自由入力だが、`python`, `claude`, `ai` など英小文字・日本語どちらも使われる。上位viewの記事タグを手動で5件確認し、`config/zenn_tags_whitelist.json` に書き出しておくと安全。

---

## 投稿時刻のAB実験

Qiita・Zennともに **月〜金の8〜9時・12時・21〜22時** にアクティブユーザーが集中する（独自計測）。ただし技術トレンドに依存するため、週ごとに時間帯を変えてview数と比較する。

```python
# scheduler.py
SLOTS = {
    "朝": "07:50",
    "昼": "11:50",
    "夜": "21:50",
}

def get_this_week_slot(week_number: int) -> str:
    keys = list(SLOTS.keys())
    key = keys[week_number % len(keys)]
    return SLOTS[key]
```

Task Schedulerに登録するコマンドは時刻パラメータを週番号で自動選択するため、人手なしで時間帯ローテーションが回る。

---

## view数の自動集計とフィードバックループ

投稿から48時間後にAPIでview数を取得し、`data/view_log.jsonl` に追記する。

```python
# view_collector.py
import requests, json, os, datetime

def collect_qiita_views(article_id: str) -> dict:
    r = requests.get(
        f"https://qiita.com/api/v2/items/{article_id}",
        headers={"Authorization": f"Bearer {os.environ['QIITA_TOKEN']}"},
    )
    item = r.json()
    return {
        "id": article_id,
        "title": item["title"],
        "tags": [t["name"] for t in item["tags"]],
        "views": item["page_views_count"],
        "collected_at": datetime.datetime.now().isoformat(),
    }

def append_log(record: dict, path: str = "data/view_log.jsonl"):
    with open(path, "a", encoding="utf-8") as f:
        f.write(json.dumps(record, ensure_ascii=False) + "\n")
```

集計後、**上位20%の勝ちパターン**（タグ・タイトル型・時間帯）を抽出して `config/winner_patterns.json` に書き出す。

```python
# winner_extractor.py
import json
from collections import Counter

def extract_winners(log_path: str = "data/view_log.jsonl") -> dict:
    records = [json.loads(l) for l in open(log_path, encoding="utf-8")]
    records.sort(key=lambda r: r["views"], reverse=True)
    top = records[: max(1, len(records) // 5)]  # 上位20%

    tag_count: Counter = Counter()
    for r in top:
        tag_count.update(r["tags"])

    return {
        "top_tags": [t for t, _ in tag_count.most_common(10)],
        "updated_at": records[0]["collected_at"] if records else "",
    }
```

次回の記事生成時、このJSONをプロンプトに注入する。

```python
winners = json.load(open("config/winner_patterns.json"))
prompt = f"""
以下のタグが高view率です: {winners['top_tags']}
このトレンドに沿ったタイトルと記事を生成してください。
"""
```

---

## 実装上の注意点

- **Qiita APIの`page_views_count`** は本人トークンでのみ取得可能。他人の記事は取れない
- 投稿直後（2時間以内）のview数は初速バイアスがあるため、**48〜72時間後**に集計するのが正確
- Zennはview数APIが非公式。`zenn.dev/api/articles/{slug}/stats` をスクレイピングするか、ダッシュボード手動確認との組み合わせで補う
- `winner_patterns.json` は週1回の更新で十分。毎日更新すると直近ノイズに引きずられる

---

このサイクルを4〜6週間回すと、タグ・タイトルのパターンが収束し、view数の底上げが安定する。量産速度を上げる前に、まずこの計測ループを先に動かしておくことが長期的な効率を決める。
