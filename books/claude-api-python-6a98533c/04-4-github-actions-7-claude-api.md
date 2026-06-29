---
title: "第4章　GitHub Actionsで毎朝7時に自動実行するClaude APIスケジューラーの実装"
free: false
---

## GitHub ActionsのCron設定とシークレット管理

GitHub Actions を使うと、サーバーを自分で用意しなくてもパイプラインを定期実行できる。まず `.github/workflows/morning_pipeline.yml` を作成し、毎朝7時（JST）に起動するジョブを定義する。GitHub ActionsのCronはUTC基準のため、JSTの7:00は `0 22 * * *`（前日22:00 UTC）と指定する。この時差換算を誤ると投稿時刻がずれるため注意が必要だ。

```yaml
name: Morning Content Pipeline

on:
  schedule:
    - cron: "0 22 * * *"   # UTC 22:00 = JST 07:00
  workflow_dispatch:         # 手動実行ボタンも残す

jobs:
  generate-and-post:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Generate content via Claude API
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          ZENN_GITHUB_PAT:   ${{ secrets.ZENN_GITHUB_PAT }}
          QIITA_TOKEN:       ${{ secrets.QIITA_TOKEN }}
        run: python src/orchestrator.py
```

APIキーは絶対にYAMLにハードコードせず、リポジトリの **Settings → Secrets and variables → Actions** から登録する。環境変数名を揃えておけば、ローカルの `.env` とGitHub Secretsを同じコードで切り替えられる。

---

## Zennへの自動投稿：Gitプッシュがそのまま公開になる仕組み

Zennはリポジトリ連携を設定すると、`articles/` 以下のMarkdownファイルをプッシュするだけで記事が公開される。Actionsワークフロー内でGitの設定とプッシュを行うのがポイントだ。

```python
# src/posters/zenn.py
import subprocess, os, textwrap
from datetime import datetime

def post_to_zenn(title: str, body: str, topics: list[str]) -> None:
    slug = datetime.now().strftime("%Y%m%d-%H%M%S")
    frontmatter = textwrap.dedent(f"""\
        ---
        title: "{title}"
        emoji: "🤖"
        type: "tech"
        topics: {topics}
        published: true
        ---
    """)
    path = f"articles/{slug}.md"
    with open(path, "w") as f:
        f.write(frontmatter + "\n" + body)

    subprocess.run(["git", "config", "user.email", "bot@example.com"], check=True)
    subprocess.run(["git", "config", "user.name",  "content-bot"],    check=True)
    subprocess.run(["git", "add", path], check=True)
    subprocess.run(["git", "commit", "-m", f"add: {title[:40]}"], check=True)

    pat = os.environ["ZENN_GITHUB_PAT"]
    remote = f"https://x-access-token:{pat}@github.com/YOUR_ORG/zenn-content.git"
    subprocess.run(["git", "push", remote, "HEAD:main"], check=True)
```

**落とし穴**：`GIT_TERMINAL_PROMPT=0` を設定しないと、PATが無効な場合にCI側でパスワード入力待ちのままハングする。環境変数として `GIT_TERMINAL_PROMPT: "0"` をジョブの `env` ブロックに追加しておくこと。

---

## Qiitaへの自動投稿：APIレート制限への対処

QiitaはREST APIで記事を投稿できるが、短時間に連投すると **429 Too Many Requests** が返る。1日の上限は60件程度のため、複数記事を生成する場合は投稿間隔を最低5時間以上空ける実装が必要だ。

```python
# src/posters/qiita.py
import os, requests, time

QIITA_API = "https://qiita.com/api/v2/items"
MIN_INTERVAL_HOURS = int(os.getenv("QIITA_MIN_INTERVAL_HOURS", "5"))

def post_to_qiita(title: str, body: str, tags: list[str]) -> dict:
    headers = {
        "Authorization": f"Bearer {os.environ['QIITA_TOKEN']}",
        "Content-Type":  "application/json",
    }
    payload = {
        "title": title,
        "body":  body,
        "tags":  [{"name": t} for t in tags],
        "private": False,
    }
    resp = requests.post(QIITA_API, json=payload, headers=headers, timeout=30)
    if resp.status_code == 429:
        raise RuntimeError(f"Qiita rate limit hit. Wait {MIN_INTERVAL_HOURS}h.")
    resp.raise_for_status()
    return resp.json()
```

タグは `["Python", "Claude", "生成AI"]` のように実在する正規タグを使う。`"AI副業"` のような独自タグは検索に引っかからずビュー数がほぼゼロになる。

---

## ワークフロー全体の動作確認

ローカルで動作を確認するには `act` コマンドが便利だが、まずは `workflow_dispatch` を使ってGitHub UI上から手動トリガーして動作を確認するのが最速だ。ログは **Actions タブ → 各ジョブ → ステップ展開** で確認できる。

生成結果がZennのリポジトリに反映されているか、Qiitaのマイページに記事が増えているかを毎朝確認する習慣をつければ、スケジューラーは安定して回り続ける。
