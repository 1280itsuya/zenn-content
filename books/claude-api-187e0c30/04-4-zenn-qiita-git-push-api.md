---
title: "第4章：Zenn・Qiitaへの自動投稿を個人開発で配線する——GitリポジトリPush型とAPI型の実装比較"
free: false
---

## 第4章：Zenn・Qiitaへの自動投稿を個人開発で配線する——GitリポジトリPush型とAPI型の実装比較

---

## ZennとQiitaで投稿方式が根本的に異なる理由

Zennはコンテンツ管理をGitHubリポジトリに委ねる設計だ。記事のMarkdownをリポジトリにpushすれば、Zennが自動的に記事を公開する。一方Qiitaは標準的なREST APIで記事の作成・更新ができる。この違いは自動化の実装方針を大きく左右する。

| 観点 | Zenn（Push型） | Qiita（API型） |
|------|--------------|--------------|
| 認証 | GitHubリポジトリ連携 | APIトークン（Bearer） |
| 投稿操作 | `git push` | `POST /api/v2/items` |
| レートリミット | GitHubのpushレート | 1,000リクエスト/時 |
| 主な落とし穴 | TTY待ちハング | 429エラー・重複投稿 |

---

## Zenn：git pushの非対話化

スクリプトからそのまま`git push`を呼ぶと、GitHub認証ダイアログがTTY入力を待ち続け、パイプラインが**2時間以上ハング**することがある。対策は2つセットで行う。

```python
# orchestrator/posters/zenn.py
import subprocess
import os

def push_to_zenn(article_path: str, commit_msg: str) -> bool:
    repo_path = os.environ["ZENN_REPO_PATH"]  # 例: /home/user/zenn-content
    pat = os.environ["GITHUB_PAT"]
    remote_url = f"https://{pat}@github.com/{os.environ['GITHUB_USER']}/zenn-content.git"

    env = os.environ.copy()
    env["GIT_TERMINAL_PROMPT"] = "0"  # 認証ダイアログを出さない

    cmds = [
        ["git", "-C", repo_path, "add", article_path],
        ["git", "-C", repo_path, "commit", "-m", commit_msg],
        ["git", "-C", repo_path, "push", remote_url, "main"],
    ]
    for cmd in cmds:
        result = subprocess.run(cmd, env=env, capture_output=True, text=True, timeout=60)
        if result.returncode != 0:
            print(f"[zenn] push失敗: {result.stderr}")
            return False
    return True
```

**重要：** `remote_url`にPATを平文で埋め込む方式はgit logに残らないが、`.env`や環境変数から漏洩しないよう注意する。リポジトリをpublicにする場合はPATをrevoke→再発行すること。

---

## Qiita：APIとレートゲートの実装

Qiitaは連続投稿すると429（Too Many Requests）を返す。特に自動化パイプラインが朝に記事を大量生成するケースでは、**5時間以上の間隔**を設けるゲートが必須だ。

```python
# orchestrator/posters/qiita.py
import os, time, json, requests
from pathlib import Path
from datetime import datetime, timedelta
from dotenv import load_dotenv

load_dotenv()

QIITA_TOKEN = os.environ["QIITA_API_TOKEN"]
MIN_INTERVAL_HOURS = float(os.getenv("QIITA_MIN_INTERVAL_HOURS", "5"))
LAST_POST_FILE = Path("data/qiita_last_post.json")

def _load_last_post() -> dict:
    if LAST_POST_FILE.exists():
        return json.loads(LAST_POST_FILE.read_text())
    return {}

def _save_last_post(item_id: str):
    data = _load_last_post()
    data["last_id"] = item_id
    data["last_time"] = datetime.now().isoformat()
    LAST_POST_FILE.write_text(json.dumps(data))

def can_post_now() -> bool:
    data = _load_last_post()
    if "last_time" not in data:
        return True
    last = datetime.fromisoformat(data["last_time"])
    return datetime.now() - last >= timedelta(hours=MIN_INTERVAL_HOURS)

def post_to_qiita(title: str, body: str, tags: list[str]) -> str | None:
    if not can_post_now():
        print("[qiita] レートゲート: スキップ（前回投稿から5時間未満）")
        return None

    headers = {"Authorization": f"Bearer {QIITA_TOKEN}", "Content-Type": "application/json"}
    payload = {
        "title": title,
        "body": body,
        "tags": [{"name": t} for t in tags],
        "private": False,
    }
    resp = requests.post("https://qiita.com/api/v2/items", json=payload, headers=headers)
    if resp.status_code == 201:
        item_id = resp.json()["id"]
        _save_last_post(item_id)
        return item_id
    print(f"[qiita] 投稿失敗 {resp.status_code}: {resp.text[:200]}")
    return None
```

`QIITA_MIN_INTERVAL_HOURS`を`.env`で調整できるようにしておくと、テスト時に`0.1`に下げて動作確認できる。

---

## 重複スキップの実装

同じタイトルの記事が複数回投稿されると、読者の信頼を失うだけでなくアカウント凍結リスクもある。タイトルのハッシュをDBに保存して事前チェックする。

```python
# orchestrator/utils/dedup.py
import hashlib, sqlite3
from pathlib import Path

DB_PATH = Path("data/posted.db")

def _conn():
    conn = sqlite3.connect(DB_PATH)
    conn.execute("CREATE TABLE IF NOT EXISTS posted (hash TEXT PRIMARY KEY, platform TEXT, posted_at TEXT)")
    return conn

def already_posted(title: str, platform: str) -> bool:
    h = hashlib.sha256(f"{platform}:{title}".encode()).hexdigest()
    with _conn() as conn:
        return conn.execute("SELECT 1 FROM posted WHERE hash=?", (h,)).fetchone() is not None

def mark_posted(title: str, platform: str):
    h = hashlib.sha256(f"{platform}:{title}".encode()).hexdigest()
    from datetime import datetime
    with _conn() as conn:
        conn.execute("INSERT OR IGNORE INTO posted VALUES (?,?,?)", (h, platform, datetime.now().isoformat()))
```

呼び出し側では投稿前に`already_posted(title, "qiita")`を確認し、`True`なら処理を中断する。

---

## 落とし穴まとめ

- **`load_dotenv()`の欠落**：`os.environ`だけでは`.env`を読まない。各ポスターファイルの先頭に必ず書く
- **Qiitaトークン失効**：403が突然返り始めたらトークン再発行が必要。エラーログに`403`を検知したら通知する仕組みを入れる
- **Zennのrebaseコンフリクト**：複数チャンネルが同じリポジトリにpushすると競合する。`git pull --rebase`を`push`前に差し込むか、チャンネルごとにブランチを分ける
