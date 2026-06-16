---
title: "パイプラインの収益計測と改善ループの回し方"
free: false
---

## パイプラインの収益計測と改善ループの回し方

コンテンツを量産しても、どの記事が稼いでいるかを把握していなければ改善できない。このセクションでは、投稿ごとのview・クリック・アフィリエイト収益をCSVに集約し、Claudeに「勝ちテーマ」を抽出させるフィードバックループを構築する。

---

## 収益データをCSVに集約する

まず各チャネルのデータを一本のCSVに統合する。Zennのview数はAPIで、A8.netの成果報酬は管理画面からCSVエクスポートして読み込む。

```python
import csv
import requests
from datetime import date

def fetch_zenn_views(username: str) -> list[dict]:
    url = f"https://zenn.dev/api/articles?username={username}&order=latest"
    articles = requests.get(url).json()["articles"]
    return [
        {
            "date": a["published_at"][:10],
            "title": a["title"],
            "slug": a["slug"],
            "views": a.get("liked_count", 0),  # Zenn公開APIはlikedのみ
            "channel": "zenn",
        }
        for a in articles
    ]

def merge_affiliate_csv(affiliate_path: str, records: list[dict]) -> list[dict]:
    slug_to_revenue = {}
    with open(affiliate_path, encoding="utf-8-sig") as f:
        for row in csv.DictReader(f):
            # A8エクスポートCSVの列名に合わせる
            slug_to_revenue[row["campaign_code"]] = float(row["commission"])
    for r in records:
        r["revenue"] = slug_to_revenue.get(r["slug"], 0.0)
    return records
```

> **落とし穴**: ZennのパブリックAPIはいいね数しか返さない。view数が欲しい場合は `zenn-cli` のローカルログを併用するか、Zenn公式の「ダッシュボードCSVエクスポート」（手動）を週次で取り込む設計にする。

---

## Claudeに勝ちテーマを分析させる

集約したCSVをClaudeに渡してパターンを抽出する。ここがフィードバックループの核心だ。

```python
import anthropic
import pandas as pd

def extract_winning_themes(csv_path: str) -> str:
    df = pd.read_csv(csv_path)
    top = df.nlargest(20, "views")[["title", "views", "revenue"]].to_csv(index=False)

    client = anthropic.Anthropic()
    message = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=[
            {
                "role": "user",
                "content": (
                    "以下は過去30日の記事パフォーマンスデータです。\n\n"
                    f"{top}\n\n"
                    "viewとrevenueが高い記事に共通するテーマ・タグ・タイトル構造を3点に絞って分析し、"
                    "次に量産すべきトピックを箇条書きで5件提案してください。"
                ),
            }
        ],
    )
    return message.content[0].text
```

返ってくる提案例：

```
【共通パターン】
- 「〇〇エラー 解決策」系のロングテールKWが高view
- Pythonタグ+初心者向け訴求が収益に直結
- タイトルに具体数値（"3ステップ"等）を入れると流入増

【次に量産すべきトピック】
1. "TypeError: unhashable type の直し方【Python初心者】"
2. "Claude API 料金を半額にするプロンプトキャッシュ術"
...
```

---

## 週次で自動改善するスケジューラー

このループを手動で回すと続かない。Task Scheduler（Windows）またはcronで週次実行する。

```python
# run_weekly_analysis.py
from pathlib import Path
from collect_metrics import fetch_zenn_views, merge_affiliate_csv
from analyze import extract_winning_themes
import csv, datetime

if __name__ == "__main__":
    records = fetch_zenn_views("your_username")
    records = merge_affiliate_csv("data/a8_export.csv", records)

    out_path = Path("data/metrics.csv")
    with open(out_path, "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=records[0].keys())
        writer.writeheader()
        writer.writerows(records)

    themes = extract_winning_themes(str(out_path))
    report_path = Path(f"data/winners_{datetime.date.today()}.md")
    report_path.write_text(themes, encoding="utf-8")
    print(f"分析完了 → {report_path}")
```

> **実例での落とし穴**: 初期は全記事のview合計が100未満のため、Claudeが「データ不足で分析不可」と返すことがある。最低でも30記事・2週間分のデータが蓄積してから本格稼働させる。それまでは分析ではなく「投稿本数を増やすフェーズ」と割り切る。

---

## 改善ループの全体像

```
投稿 → view/収益計測(CSV) → Claude分析 → 勝ちテーマ抽出
  ↑                                              ↓
  └─────────── 次の記事生成プロンプトへ反映 ──────┘
```

このループを週次で回すだけで、根拠のないテーマ選定から脱却できる。収益が動き始めるのは通常3〜6週目以降だが、データの蓄積自体がパイプラインの資産になる。
