---
title: "pandas 7日移動平均グラフ + Slack Incoming Webhookで月¥0のZenn KPIダッシュボードを完成させる"
free: false
---

Zennの章を執筆します。pandas/matplotlib/Slack/GitHub Actionsのコードを組み合わせた実装章です。

---

## pandas rolling(7) でview/like/follower の7日移動平均を3行追加する

前章でコミットした `data/kpi.csv` を読み込み、ノイズを除去した移動平均系列を生成する。`min_periods=1` を付けないと先頭6行がすべて `NaN` になるため必須。

```python
# src/kpi_graph.py
import pandas as pd

def load_kpi(path: str = "data/kpi.csv") -> pd.DataFrame:
    df = pd.read_csv(path, parse_dates=["date"]).sort_values("date").reset_index(drop=True)
    for col in ["views", "likes", "followers"]:
        df[f"{col}_ma7"] = df[col].rolling(7, min_periods=1).mean()
    return df
```

`rolling(7).mean()` は過去7日の単純移動平均。週次の曜日バイアス（月曜が高い等）も平滑化され、本当のトレンドだけが残る。

---

## matplotlib で折れ線グラフを PNG 生成し `graph/` ディレクトリへ保存する

生データ（薄い線）と7日MA（太い線）を重ねることで、瞬間スパイクとトレンドを同時に視覚化する。

```python
import matplotlib.pyplot as plt
import matplotlib.dates as mdates
import pathlib

def generate_graph(df: pd.DataFrame, out_dir: str = "graph") -> str:
    pathlib.Path(out_dir).mkdir(exist_ok=True)
    out_path = f"{out_dir}/views_ma7.png"

    fig, ax = plt.subplots(figsize=(10, 4), dpi=120)
    ax.plot(df["date"], df["views"], alpha=0.3, color="#4A90D9", label="raw views")
    ax.plot(df["date"], df["views_ma7"], linewidth=2.5, color="#4A90D9", label="7日MA")
    ax.xaxis.set_major_formatter(mdates.DateFormatter("%m/%d"))
    ax.xaxis.set_major_locator(mdates.WeekdayLocator(interval=1))
    fig.autofmt_xdate()
    ax.legend(fontsize=9)
    ax.set_title("Zenn views — 7-day moving average", fontsize=11)
    fig.tight_layout()
    fig.savefig(out_path)
    plt.close(fig)
    return out_path
```

`plt.close(fig)` を忘れると Actions の長時間ループでメモリリークする。

---

## `git push` で PNG を GitHub へコミットし Raw URL を確定させる

Slack に送る画像は GitHub Raw URLで配信する。`stefanzweifel/git-auto-commit-action` を使う場合はスキップしてよいが、スクリプト単体でも動くよう Python 側にも書いておく。

```python
import subprocess, datetime

REPO_RAW = "https://raw.githubusercontent.com/{OWNER}/{REPO}/main"

def commit_graph(path: str) -> str:
    today = datetime.date.today().isoformat()
    subprocess.run(["git", "config", "user.email", "bot@local"], check=True)
    subprocess.run(["git", "config", "user.name", "kpi-bot"], check=True)
    subprocess.run(["git", "add", path], check=True)
    result = subprocess.run(["git", "diff", "--cached", "--quiet"])
    if result.returncode != 0:  # 変更ありの場合のみ commit
        subprocess.run(["git", "commit", "-m", f"kpi: graph {today}"], check=True)
        subprocess.run(["git", "push"], check=True)
    return f"{REPO_RAW}/{path}"
```

差分がない日に `git commit` を呼ぶと `returncode=1` で Actions が落ちるため、`git diff --cached --quiet` でガードする。

---

## Slack Incoming Webhook に画像 URL + 前日差分を POST する

Slack の `attachments` ではなく `blocks` を使うことで、画像プレビューと数値差分を1メッセージにまとめられる。

```python
import os, requests

def notify_slack(df: pd.DataFrame, graph_url: str) -> None:
    latest = df.iloc[-1]
    prev   = df.iloc[-2] if len(df) >= 2 else latest

    dv = int(latest["views"]     - prev["views"])
    dl = int(latest["likes"]     - prev["likes"])
    df_ = int(latest["followers"] - prev["followers"])

    text = (
        f"*Zenn KPI {latest['date'].strftime('%Y-%m-%d')}*\n"
        f"view `{dv:+d}`  like `{dl:+d}`  follower `{df_:+d}`\n"
        f"7日MA view: `{latest['views_ma7']:.1f}`\n"
        f"<{graph_url}|グラフを開く>"
    )
    webhook = os.environ["SLACK_WEBHOOK_URL"]
    requests.post(webhook, json={"text": text}, timeout=10).raise_for_status()
```

Webhook URL は Slack App の「Incoming Webhooks」から発行し、GitHub Secrets に `SLACK_WEBHOOK_URL` として登録する。URLをコードにハードコードすると public リポジトリ公開時に即失効するため注意。

---

## GitHub Actions cron で毎朝 7:00 JST に全パイプラインを自動実行する

```yaml
# .github/workflows/zenn_kpi.yml
name: zenn-kpi-daily

on:
  schedule:
    - cron: "0 22 * * *"   # UTC 22:00 = JST 07:00
  workflow_dispatch:        # 手動テスト用

permissions:
  contents: write

jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.PAT }}   # push 権限付き PAT

      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: pip

      - run: pip install -r requirements.txt

      - name: Run KPI pipeline
        env:
          ZENN_COOKIE:      ${{ secrets.ZENN_COOKIE }}
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
          GITHUB_OWNER:     ${{ github.repository_owner }}
          GITHUB_REPO:      ${{ github.event.repository.name }}
        run: python src/kpi_pipeline.py

      - uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: "kpi: auto update [skip ci]"
          file_pattern: "data/*.csv graph/*.png"
```

`[skip ci]` を commit メッセージに入れることで、bot の push が再度 Actions を起動するループを防ぐ。

---

## リポジトリを public にして `.env.example` 付きで OSS 公開する

他者が `git clone → .env 埋め → 即動作` できる状態を作る最小手順。

```bash
# 1. GitHub CLI でリポジトリを public に変更
gh repo edit --visibility public --accept-visibility-change-consequences

# 2. .env から値を除いたサンプルファイルを生成
grep -E "^[A-Z_]+" .env | sed 's/=.*/=YOUR_VALUE_HERE/' > .env.example
git add .env.example && git commit -m "docs: add .env.example"

# 3. README に Secrets 設定手順を追記（最低限の項目）
cat >> README.md << 'EOF'

## Quick Start
```
git clone https://github.com/YOUR/zenn-kpi-tracker
cd zenn-kpi-tracker
cp .env.example .env  # 値を埋める
pip install -r requirements.txt
python src/kpi_pipeline.py
```

### GitHub Secrets に登録する3つ
| Key | 取得場所 |
|-----|---------|
| `ZENN_COOKIE` | 第2章手順で取得した Cookie 文字列 |
| `SLACK_WEBHOOK_URL` | Slack App > Incoming Webhooks |
| `PAT` | GitHub > Settings > Developer settings > PAT (repo スコープ) |
EOF

git add README.md && git commit -m "docs: quick start guide"
git push
```

public 化の前に `git log --all -- '*.env'` を実行し、過去コミットに Cookie や PAT が混入していないか確認すること。混入が見つかった場合は `git filter-repo --path .env --invert-paths` で履歴から除去してから public にする。

---

これで `python src/kpi_pipeline.py` の1コマンドが「Cookie取得 → CSV追記 → グラフ生成 → git push → Slack通知」を直列実行し、毎朝7時にダッシュボードが Slack に届く状態が完成する。GitHub Actions の Secrets 3つを埋めるだけで他者も同じパイプラインを複製できる。
