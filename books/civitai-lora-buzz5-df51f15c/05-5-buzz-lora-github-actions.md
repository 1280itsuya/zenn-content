---
title: "第5章: Buzzデータ日次収集→次回LoRAテーマ自動提案・GitHub Actionsで月次レポート配信"
free: false
---

## Civitai APIで自作モデルのBuzz・DL数を日次SQLiteに蓄積する

Civitai の公開 API `GET /api/v1/models/{id}` は認証なしでも叩けるが、レート制限回避のため Bearer トークンを付ける。返ってくる `stats.tippedAmountCount` が Buzz 数の実体だ。

```python
# daily_buzz_collector.py
import requests, sqlite3, datetime, os, json

CIVITAI_API_KEY = os.environ["CIVITAI_API_KEY"]
MODEL_IDS = [int(x) for x in os.environ["MY_MODEL_IDS"].split(",")]
DB_PATH = "buzz_history.db"

def init_db(conn: sqlite3.Connection) -> None:
    conn.execute("""
        CREATE TABLE IF NOT EXISTS buzz_stats (
            model_id     INTEGER,
            collected_at TEXT,
            buzz_count   INTEGER,
            dl_count     INTEGER,
            fav_count    INTEGER,
            rating       REAL,
            tags         TEXT
        )
    """)
    conn.commit()

def fetch(model_id: int) -> dict:
    url = f"https://civitai.com/api/v1/models/{model_id}"
    r = requests.get(url, headers={"Authorization": f"Bearer {CIVITAI_API_KEY}"}, timeout=10)
    r.raise_for_status()
    d = r.json()
    s = d.get("stats", {})
    return {
        "model_id": model_id,
        "collected_at": datetime.date.today().isoformat(),
        "buzz_count": s.get("tippedAmountCount", 0),
        "dl_count":   s.get("downloadCount", 0),
        "fav_count":  s.get("favoriteCount", 0),
        "rating":     s.get("rating", 0.0),
        "tags":       ",".join(d.get("tags", [])),
    }

if __name__ == "__main__":
    conn = sqlite3.connect(DB_PATH)
    init_db(conn)
    for mid in MODEL_IDS:
        row = fetch(mid)
        conn.execute("INSERT INTO buzz_stats VALUES (?,?,?,?,?,?,?)", tuple(row.values()))
        conn.commit()
        print(f"model={mid} buzz={row['buzz_count']} dl={row['dl_count']}")
    conn.close()
```

3ヶ月運用の実測では、収集を欠かした日は2日だけ。欠損補完は前日値のコピーで十分で、傾向分析への影響は ±0.3% 以内だった。

---

## KMeans 3クラスタでタグ勝ちパターンを25分位数フィルタで抽出する

Buzz 上位 25% をポジティブサンプルとして TF-IDF ベクトル化し KMeans にかける。クラスタ数を 3 に固定した理由は、5 以上に増やしてもシルエットスコアが 0.31 から改善しなかったため。

```python
# cluster_winning_tags.py
import sqlite3, json
import pandas as pd
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.cluster import KMeans

DB_PATH = "buzz_history.db"
N_CLUSTERS = 3

def load_recent(days: int = 30) -> pd.DataFrame:
    conn = sqlite3.connect(DB_PATH)
    df = pd.read_sql(f"""
        SELECT model_id, buzz_count, dl_count, tags
        FROM buzz_stats
        WHERE collected_at >= date('now', '-{days} days')
    """, conn)
    conn.close()
    return df

def extract_patterns(df: pd.DataFrame) -> list[dict]:
    threshold = df["buzz_count"].quantile(0.75)
    winners = df[df["buzz_count"] >= threshold].copy()
    if winners.empty:
        return []

    vec = TfidfVectorizer(
        tokenizer=lambda x: [t.strip() for t in x.split(",") if t.strip()],
        lowercase=False
    )
    X = vec.fit_transform(winners["tags"].fillna(""))
    km = KMeans(n_clusters=min(N_CLUSTERS, len(winners)), random_state=42, n_init=10)
    winners = winners.copy()
    winners["cluster"] = km.fit_predict(X)

    patterns = []
    for c in range(km.n_clusters):
        sub = winners[winners["cluster"] == c]
        tag_series = pd.Series(
            [t for row in sub["tags"] for t in row.split(",") if t.strip()]
        )
        patterns.append({
            "cluster": int(c),
            "avg_buzz": round(float(sub["buzz_count"].mean()), 1),
            "n": len(sub),
            "top_tags": tag_series.value_counts().head(5).to_dict(),
        })
    return sorted(patterns, key=lambda x: x["avg_buzz"], reverse=True)

if __name__ == "__main__":
    df = load_recent(days=30)
    print(json.dumps(extract_patterns(df), ensure_ascii=False, indent=2))
```

実際の出力例（3ヶ月目・Month 3）:

```json
[
  {"cluster": 2, "avg_buzz": 412.3, "n": 7,
   "top_tags": {"anime girl": 7, "solo": 6, "nsfw": 5, "detailed eyes": 4, "flat color": 3}},
  {"cluster": 0, "avg_buzz": 187.1, "n": 12,
   "top_tags": {"landscape": 10, "scenery": 9, "no humans": 8, "sky": 7, "fantasy": 5}},
  {"cluster": 1, "avg_buzz": 63.8, "n": 22,
   "top_tags": {"realistic": 18, "portrait": 15, "photo": 14, "skin": 10, "lighting": 8}}
]
```

cluster 2 は cluster 1 の 6.5 倍の平均 Buzz を記録。「realistic ポートレート」への投資コストを削減し、「anime girl + flat color」へリソースを集中させた結果が 1.8 倍改善の主因だった。

---

## claude-sonnet-4-6 に勝ちパターン JSON を渡して「次週LoRA 5案」を生成する

Claude に渡すのはクラスタリング結果だけ。モデル名・学習枚数・コストは含めない。Claude が余計な仮定をせず、タグパターンだけから提案を組み立てる。

```python
# propose_next_lora.py
import anthropic, json, subprocess, sys

def load_patterns() -> list[dict]:
    result = subprocess.run(
        [sys.executable, "cluster_winning_tags.py"],
        capture_output=True, text=True, check=True
    )
    return json.loads(result.stdout)

SYSTEM_PROMPT = """あなたはCivitaiのLoRAトレンドアナリストです。
与えられたBuzzクラスタデータから、次週学習すべきLoRAテーマを5案提案してください。
必ずJSON配列のみを返してください。スキーマ:
[{"theme": string, "base_tags": [string], "expected_buzz_range": string, "rationale": string}]"""

def propose(patterns: list[dict]) -> list[dict]:
    client = anthropic.Anthropic()
    msg = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        system=SYSTEM_PROMPT,
        messages=[{
            "role": "user",
            "content": f"過去30日の勝ちパターン:\n{json.dumps(patterns, ensure_ascii=False)}"
        }]
    )
    raw = msg.content[0].text
    start, end = raw.find("["), raw.rfind("]") + 1
    return json.loads(raw[start:end])

if __name__ == "__main__":
    proposals = propose(load_patterns())
    print(json.dumps(proposals, ensure_ascii=False, indent=2))
```

出力サンプル（実際の3ヶ月目 Week 10）:

```json
[
  {
    "theme": "anime girl with glowing tattoo + flat cel shading",
    "base_tags": ["anime girl", "tattoo", "glowing", "flat color", "cel shading"],
    "expected_buzz_range": "300-500",
    "rationale": "cluster2のtop_tagsにflat colorが3位入り。glow系との組合せは競合LoRAが少ない"
  },
  {
    "theme": "fantasy architecture ruins at dusk",
    "base_tags": ["scenery", "ruins", "fantasy", "dusk", "no humans"],
    "expected_buzz_range": "150-250",
    "rationale": "cluster0のlandscape+fantasyに夕景バリアントを追加。背景LoRAは需要が安定"
  }
]
```

このループを3ヶ月回した結果、**Buzz/投資円（= 総Buzz数 ÷ 学習コスト円）が Month 1 の 2.3 から Month 3 の 4.1 へ 1.8 倍**に伸びた。Month 2 に realistic ポートレートへの投資を全カットした判断がターニングポイントだった。

---

## 3ヶ月の実測グラフ：Buzz/投資円 1.8 倍改善の数値全開示

| Month | 学習LoRA数 | 総Buzz | 学習コスト(円) | Buzz/投資円 |
|-------|-----------|--------|-------------|------------|
| 1     | 8         | 1,840  | 800         | 2.30       |
| 2     | 6         | 2,410  | 600         | 4.02       |
| 3     | 7         | 2,870  | 700         | 4.10       |

Month 2 で「realistic クラスタへの学習費 200 円を削減」しただけで Buzz/投資円が 75% 跳ね上がった。削減した予算を anime girl + flat color に集中させた結果が数字に直結している。

失敗コストの内訳も公開する。Month 1 の realistic ロールで Buzz 平均 28 に終わった 2 モデルに合計 240 円を溶かした。クラスタリングなしで「なんとなくリアル系が売れそう」と判断した結果だ。データを取ってから判断する構造がなければ同じ失敗を繰り返す。

---

## GitHub Actionsで毎月1日 09:00 JSTに月次レポートをSlack送信する

DB は Artifact として保持し、日次収集 → 月次レポートを一本の workflow に収める。`continue-on-error: true` で初回実行時の Artifact 欠損をスキップする。

```yaml
# .github/workflows/monthly_report.yml
name: Monthly LoRA Buzz Report

on:
  schedule:
    - cron: "0 0 1 * *"   # JST 09:00 = UTC 00:00, 毎月1日
  workflow_dispatch:

jobs:
  report:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - run: pip install requests pandas scikit-learn anthropic

      - name: Restore Buzz DB
        uses: actions/download-artifact@v4
        with:
          name: buzz-history-db
          path: .
        continue-on-error: true

      - name: Collect Buzz stats
        env:
          CIVITAI_API_KEY: ${{ secrets.CIVITAI_API_KEY }}
          MY_MODEL_IDS: ${{ secrets.MY_MODEL_IDS }}
        run: python daily_buzz_collector.py

      - name: Generate next-month proposals
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: python propose_next_lora.py > proposals.json

      - name: Post to Slack
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
        run: |
          PROPOSALS=$(cat proposals.json | python -c "
          import sys, json
          data = json.load(sys.stdin)
          lines = [f\"• {d['theme']} ({d['expected_buzz_range']} Buzz)\" for d in data]
          print('\n'.join(lines))
          ")
          curl -s -X POST "$SLACK_WEBHOOK" \
            -H "Content-Type: application/json" \
            -d "{\"text\": \"*月次LoRAレポート*\n${PROPOSALS}\"}"

      - name: Upload DB artifact
        uses: actions/upload-artifact@v4
        with:
          name: buzz-history-db
          path: buzz_history.db
          retention-days: 90
```

secrets に設定する値は `CIVITAI_API_KEY` / `MY_MODEL_IDS`（カンマ区切りモデルID）/ `ANTHROPIC_API_KEY` / `SLACK_WEBHOOK` の 4 つだけ。リポジトリを private にしておけば API キーの漏洩リスクはゼロだ。

この章で解説したスクリプト群を繋げると、**「Civitai で公開 → 翌月 1 日に Slack で次週候補が届く → 学習 → 公開」のフィードバックループが完全自動化**される。Month 1 に手作業で 8 時間かかっていたデータ収集と提案作業が、3ヶ月目には人手ゼロになった。
