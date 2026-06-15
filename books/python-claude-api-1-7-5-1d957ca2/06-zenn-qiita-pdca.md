---
title: "ZennとQiitaへの自動投稿で流入を計測しPDCAを回す"
free: false
---

## ZennとQiitaへの自動投稿で流入を計測しPDCAを回す

記事を生成しても投稿しなければ何も始まらない。さらに投稿しただけでは「何が読まれているか」がわからず、生成精度を上げられない。本章では**投稿→計測→抽出→生成改善**のループを全自動で回す実装を解説する。

---

## Zennはgit pushで投稿する

Zennには投稿APIがない。代わりにGitHubリポジトリと連携させ、`git push`をトリガーにする。

```
zenn-content/
  articles/
    2026-06-15-affiliate-auto.md
```

```python
# posters/zenn.py
import subprocess, os
from dotenv import load_dotenv
load_dotenv()

def publish_zenn(article_path: str, title: str, topics: list[str]):
    # Markdownフロントマターを付与
    front = f"""---
title: "{title}"
emoji: "🤖"
type: "tech"
topics: {topics}
published: true
---
"""
    dest = f"zenn-content/articles/{article_path}.md"
    content = open(article_path).read()
    with open(dest, "w", encoding="utf-8") as f:
        f.write(front + content)

    repo = os.environ["ZENN_GITHUB_REPO_DIR"]
    pat  = os.environ["GITHUB_PAT"]
    subprocess.run(["git", "-C", repo, "add", "."], check=True)
    subprocess.run(["git", "-C", repo, "commit", "-m", f"add: {title}"], check=True)
    # 非対話化必須。TTYが無いと認証ダイアログでハングする
    remote = f"https://{pat}@github.com/yourname/zenn-content.git"
    subprocess.run(
        ["git", "-C", repo, "push", remote, "main"],
        check=True,
        env={**os.environ, "GIT_TERMINAL_PROMPT": "0"}
    )
```

**落とし穴**: `GIT_TERMINAL_PROMPT=0`を渡さないとTask Scheduler実行時にパスワードプロンプト待ちで無限ハングする。また`published: true`を忘れると下書きのまま公開されず、view数も取れない。

---

## QiitaはAPIで直接投稿する

```python
# posters/qiita.py
import requests, time, os
from dotenv import load_dotenv
load_dotenv()

QIITA_TOKEN = os.environ["QIITA_TOKEN"]
MIN_INTERVAL_HOURS = float(os.environ.get("QIITA_MIN_INTERVAL_HOURS", "5"))

def publish_qiita(title: str, body: str, tags: list[str]) -> str | None:
    # 最低間隔チェック（429対策）
    last_path = ".qiita_last_post"
    if os.path.exists(last_path):
        elapsed = (time.time() - os.path.getmtime(last_path)) / 3600
        if elapsed < MIN_INTERVAL_HOURS:
            print(f"Qiita: {elapsed:.1f}h < {MIN_INTERVAL_HOURS}h, skip")
            return None

    tag_objs = [{"name": t, "versions": []} for t in tags[:5]]
    resp = requests.post(
        "https://qiita.com/api/v2/items",
        headers={"Authorization": f"Bearer {QIITA_TOKEN}"},
        json={"title": title, "body": body, "tags": tag_objs, "private": False},
        timeout=30,
    )
    resp.raise_for_status()
    open(last_path, "w").close()
    return resp.json()["id"]
```

タグは`Python`・`AI`・`自動化`のように実在する正規タグを使う。ニュース由来のキーワードをそのままタグにすると検索に引っかからず、viewがゼロになる。

---

## view数を毎日収集する

```python
# src/winner_extractor.py
import requests, json, os
from dotenv import load_dotenv
load_dotenv()

def fetch_qiita_views() -> list[dict]:
    token = os.environ["QIITA_TOKEN"]
    items = []
    page = 1
    while True:
        r = requests.get(
            "https://qiita.com/api/v2/authenticated_user/items",
            headers={"Authorization": f"Bearer {token}"},
            params={"page": page, "per_page": 20},
        ).json()
        if not r: break
        items += [{"id": i["id"], "title": i["title"],
                   "views": i.get("page_views_count", 0),
                   "tags": [t["name"] for t in i["tags"]]} for i in r]
        page += 1
    return sorted(items, key=lambda x: x["views"], reverse=True)

def extract_winners(top_n=5) -> dict:
    items = fetch_qiita_views()
    winners = items[:top_n]
    # 勝ちタグを集計
    tag_freq: dict[str, int] = {}
    for item in winners:
        for tag in item["tags"]:
            tag_freq[tag] = tag_freq.get(tag, 0) + 1
    best_tags = sorted(tag_freq, key=tag_freq.get, reverse=True)[:5]

    result = {"top_articles": winners, "winning_tags": best_tags}
    os.makedirs("data", exist_ok=True)
    with open("data/winners.json", "w", encoding="utf-8") as f:
        json.dump(result, f, ensure_ascii=False, indent=2)
    return result
```

ZennはAPIがなくスクレイピングも利用規約上グレーなため、Zennのview計測は手動確認にとどめ、自動計測はQiitaに絞るのが現実的だ。

---

## 勝ちテーマを生成へフィードバックする

```python
# src/topic_strategist.py
import json, anthropic

def propose_next_topics() -> list[str]:
    with open("data/winners.json", encoding="utf-8") as f:
        winners = json.load(f)

    top_titles  = [a["title"] for a in winners["top_articles"]]
    winning_tags = winners["winning_tags"]

    client = anthropic.Anthropic()
    prompt = f"""
以下はQiitaで最もview数が多い記事タイトルと頻出タグです。
タイトル: {top_titles}
頻出タグ: {winning_tags}

この傾向を踏まえ、次に書くべき具体的な記事タイトルを5本提案してください。
読者はPythonを使う個人開発者。タイトルは「〇〇でXXを△△する方法」形式。
JSON配列で返してください。
"""
    msg = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=512,
        messages=[{"role": "user", "content": prompt}],
    )
    return json.loads(msg.content[0].text)
```

このループを毎朝7時のスケジューラに組み込む。

```
07:00 generate_articles.py  ← winners.jsonを参照してClaude生成
07:30 posters/qiita.py      ← 投稿
翌03:00 winner_extractor.py ← view集計・winners.json更新
```

---

## 実例：1週間で見えた傾向

実際に10本投稿したところ、「エラー文をそのままタイトルに入れた記事」がview上位を独占した。`ModuleNotFoundError: No module named 'dotenv'`という記事は3日で83viewを記録し、抽象的なハウツー記事（「Pythonで自動化する方法」など）はほぼゼロだった。

`winner_extractor.py`がこの傾向を`data/winners.json`に書き出し、翌朝の`topic_strategist.py`が「実在エラー文1件＝1記事」という戦略に自動でシフトした。人間が方針を変えるより一周早く、パイプラインが自分で学習する形になる。
