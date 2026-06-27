---
title: "自動投稿パイプライン構築──Zenn/Qiita APIとGitHub Actionsで毎朝7時公開"
free: false
---

## 自動投稿パイプライン構築──Zenn/Qiita APIとGitHub Actionsで毎朝7時公開

### API制限と認証の前提知識

ZennはREST APIを公開していないため、記事公開は**GitHubリポジトリへのpush**で行う。`zenn-cli`が管理するMarkdownファイルをGitHub上の指定リポジトリに置くと、Zennが自動同期する仕組みだ。一方QiitaはRESTful APIを持ち、`POST /api/v2/items`で記事を直接投稿できるが、1時間あたり60リクエストのレート制限がある。複数記事を一気に投稿するとすぐに429エラーが返るため、**投稿間隔を最低5分以上**確保する必要がある。

```python
import time, requests

QIITA_TOKEN = os.environ["QIITA_TOKEN"]
HEADERS = {"Authorization": f"Bearer {QIITA_TOKEN}"}

def post_to_qiita(title: str, body: str, tags: list[str]) -> dict:
    payload = {
        "title": title,
        "body": body,
        "tags": [{"name": t} for t in tags],
        "private": False,
    }
    resp = requests.post(
        "https://qiita.com/api/v2/items",
        json=payload,
        headers=HEADERS,
    )
    resp.raise_for_status()
    time.sleep(300)  # 5分待機でレートゲート
    return resp.json()
```

---

### タグ正規化の落とし穴

Qiitaのタグは**大文字小文字・スペースが完全一致**で管理される。`python`と`Python`は別タグ扱いになり、存在しないタグを指定すると記事が孤立してview数がゼロになる。LLMに自由生成させると必ず不正タグが混入するため、許可リストでフィルタリングする。

```python
VALID_QIITA_TAGS = {
    "Python", "JavaScript", "TypeScript", "Claude", "OpenAI",
    "機械学習", "自動化", "GitHub", "Docker", "AWS",
}

def normalize_tags(raw: list[str], limit: int = 5) -> list[str]:
    matched = [t for t in raw if t in VALID_QIITA_TAGS]
    if not matched:
        matched = ["Python"]  # フォールバック
    return matched[:limit]
```

Zennでは `topics` フィールドに同様の制約がある。`zenn.dev/topics`で公式リストを確認し、同じ許可リストを用意しておくこと。

---

### Git push の非対話化──最大の落とし穴

Zennの自動投稿で最も詰まるのがGit認証だ。`git push`を素直に実行するとGitHubが対話ダイアログを開き、CIや夜間バッチがそこで**数時間ハング**する。

解決策は2つある。

**① Personal Access Token(PAT)をURLに埋め込む**

```python
import subprocess, os

def push_zenn_article(repo_path: str) -> None:
    pat = os.environ["GITHUB_PAT"]
    remote = f"https://{pat}@github.com/yourname/zenn-content.git"
    env = {**os.environ, "GIT_TERMINAL_PROMPT": "0"}
    subprocess.run(
        ["git", "-C", repo_path, "push", remote, "main"],
        env=env,
        check=True,
        capture_output=True,
    )
```

`GIT_TERMINAL_PROMPT=0`を必ずセットする。これを忘れると、PAT埋め込みが失敗したときに対話モードへフォールバックしてハングする。

**② GitHub Actions内では`GITHUB_TOKEN`を使う**

Actions内では`secrets.GITHUB_TOKEN`が自動発行されるため、PATは不要だ。`git remote set-url`でURLを差し替えるだけで動く。

---

### GitHub Actions で毎朝7時に定時実行

```yaml
# .github/workflows/auto-post.yml
name: Auto Post Pipeline

on:
  schedule:
    - cron: "0 22 * * *"   # UTC 22:00 = JST 07:00
  workflow_dispatch:        # 手動起動も可能

jobs:
  post:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: yourname/zenn-content
          token: ${{ secrets.GITHUB_TOKEN }}
          path: zenn-content

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Generate and post articles
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          QIITA_TOKEN: ${{ secrets.QIITA_TOKEN }}
          GITHUB_PAT: ${{ secrets.PAT_FOR_ZENN }}
        run: python src/orchestrator.py

      - name: Push Zenn articles
        run: |
          cd zenn-content
          git config user.email "bot@example.com"
          git config user.name "AutoBot"
          git add -A
          git diff --cached --quiet || git commit -m "auto: $(date '+%Y-%m-%d')"
          git push
```

`cron: "0 22 * * *"`はUTC基準のため、JST 07:00に相当する。この時差換算は定番の落とし穴で、`"0 7 * * *"`と書いてしまうとJST 16:00に実行される。

`secrets`にはGitHubのリポジトリ設定画面（Settings → Secrets and variables → Actions）から登録する。ローカルの`.env`ファイルに書いたPATをそのままコミットするのは絶対に避けること。一度でも平文でpushするとGitHubのシークレットスキャンが検知し、トークンを即座に失効させる。

---

### 動作確認の手順

1. `workflow_dispatch`トリガーで手動実行し、Actionsのログで各ステップのexit codeを確認する
2. QiitaのAPIレスポンスに含まれる`url`フィールドをログ出力し、実際に記事が公開されているか目視する
3. Zennは同期に最大2〜3分かかるため、push後すぐ確認しても反映前の場合がある

この2プラットフォームが安定して動き始めれば、同じ構造でDev.toやはてなブログも追加できる。APIの形式が異なるだけで、「生成→正規化→投稿→push」という骨格は共通だ。
