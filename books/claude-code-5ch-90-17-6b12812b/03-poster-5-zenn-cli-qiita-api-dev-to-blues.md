---
title: "Poster実装5選：zenn-cli・Qiita API・Dev.to・Bluesky ATP・note CDPの差異と認証設定"
free: false
---

5つのポスター実装を並べると認証フロー・レート制限・タグ仕様がそれぞれ異なり、1つでも設定漏れがあると**無音で投稿されない**か**2時間ブロック**になる。実測した落とし穴を全件コードで示す。

---

## zenn-cli：GIT_TERMINAL_PROMPT=0 未設定で2時間ブロック

`git push` がTTY認証ダイアログを待ち続け、タイムアウトまで約2時間プロセスが占有される（実測）。`load_dotenv()` の欠落もPAT未読込みによる別の認証失敗を引き起こす。

```python
# posters/zenn.py
import os, subprocess
from dotenv import load_dotenv

load_dotenv()  # 欠落するとPATが読まれずサイレント失敗

def push_to_zenn(article_path: str) -> bool:
    pat = os.getenv("ZENN_GITHUB_PAT")
    repo = os.getenv("ZENN_REPO")
    if not (pat and repo):
        return False  # .env未設定→スキップ

    remote = f"https://{pat}@github.com/{repo}.git"
    env = {**os.environ, "GIT_TERMINAL_PROMPT": "0", "GIT_ASKPASS": "echo"}

    subprocess.run(["git", "remote", "set-url", "origin", remote], env=env, check=True)
    subprocess.run(["git", "add", article_path], env=env, check=True)
    subprocess.run(["git", "commit", "-m", f"add: {article_path}"], env=env, check=True)
    result = subprocess.run(["git", "push", "origin", "main"],
                            env=env, timeout=30, capture_output=True)
    return result.returncode == 0
```

## Qiita API：1日20本上限と5時間レートゲート

上限超過で `403` が返り、その後の全リクエストも一時拒否される。poster側に**最低5時間インターバル**のレートゲートを持たせ、orchestratorに依存しない独立した制御にする。

```python
# posters/qiita.py
import time, os, httpx
from pathlib import Path
from dotenv import load_dotenv

load_dotenv()
RATE_FILE = Path(".last_qiita_post")
MIN_INTERVAL = int(os.getenv("QIITA_MIN_INTERVAL_HOURS", "5")) * 3600

def _rate_ok() -> bool:
    if not RATE_FILE.exists():
        return True
    return (time.time() - RATE_FILE.stat().st_mtime) >= MIN_INTERVAL

def post_to_qiita(title: str, body: str, tags: list[str]) -> bool:
    token = os.getenv("QIITA_TOKEN")
    if not token or not _rate_ok():
        return False

    resp = httpx.post(
        "https://qiita.com/api/v2/items",
        headers={"Authorization": f"Bearer {token}"},
        json={"title": title, "body": body,
              "tags": [{"name": t} for t in tags[:5]], "private": False},
        timeout=15,
    )
    if resp.status_code == 201:
        RATE_FILE.touch()
    return resp.status_code == 201
```

## Dev.to：4スラッグ必須・未整合タグで記事が非表示

Dev.toはタグがスラッグ形式（小文字・ハイフン区切り）かつ**最大4個**。未登録スラッグを含む記事は `201 Created` が返るにもかかわらず一覧に表示されない。正規化フィルタを必ず挟む。

```python
# posters/devto.py
import os, httpx
from dotenv import load_dotenv

load_dotenv()

KNOWN_TAGS = {
    "python", "javascript", "ai", "webdev", "programming",
    "typescript", "tutorial", "opensource", "productivity", "beginners"
}

def normalize_devto_tags(raw: list[str]) -> list[str]:
    slugs = [t.lower().replace(" ", "-").replace("_", "-") for t in raw]
    valid = [s for s in slugs if s in KNOWN_TAGS]
    return (valid or ["programming"])[:4]

def post_to_devto(title: str, body: str, tags: list[str]) -> bool:
    key = os.getenv("DEVTO_API_KEY")
    if not key:
        return False

    resp = httpx.post(
        "https://dev.to/api/articles",
        headers={"api-key": key},
        json={"article": {
            "title": title, "body_markdown": body,
            "tags": normalize_devto_tags(tags), "published": True
        }},
        timeout=15,
    )
    return resp.status_code == 201
```

## Bluesky ATP：DIDトークン取得フロー（OAuth非対応）

BlueskyはOAuthでなく**AT Protocol独自のDID認証**。`createSession` でJWTを取得してから投稿する。セッション有効期限は約2時間のため、毎投稿時に再取得するのが安全。

```python
# posters/bluesky.py
import os, httpx
from datetime import datetime, timezone
from dotenv import load_dotenv

load_dotenv()
BSKY = "https://bsky.social/xrpc"

def _session() -> dict:
    r = httpx.post(f"{BSKY}/com.atproto.server.createSession",
                   json={"identifier": os.getenv("BLUESKY_HANDLE"),
                         "password": os.getenv("BLUESKY_PASSWORD")})
    r.raise_for_status()
    return r.json()  # {"did": "...", "accessJwt": "..."}

def post_to_bluesky(text: str) -> bool:
    if not (os.getenv("BLUESKY_HANDLE") and os.getenv("BLUESKY_PASSWORD")):
        return False

    s = _session()
    r = httpx.post(
        f"{BSKY}/com.atproto.repo.createRecord",
        headers={"Authorization": f"Bearer {s['accessJwt']}"},
        json={"repo": s["did"], "collection": "app.bsky.feed.post",
              "record": {"$type": "app.bsky.feed.post",
                         "text": text[:300],
                         "createdAt": datetime.now(timezone.utc).isoformat()}},
        timeout=15,
    )
    return r.status_code == 200
```

## note CDP：Playwright SPA操作と1記事¥30換算コスト

noteはパブリックAPIが存在しない（2026年6月時点）。Playwrightでログイン→新規作成→公開の3画面を遷移する。Claude Computer Use APIで代替する場合、**入力8k tokens換算で1記事¥30〜¥60のAPIコスト**が発生するため、本数を絞るか月次コスト上限を設定すること。

```python
# posters/note_cdp.py
import os
from playwright.sync_api import sync_playwright
from dotenv import load_dotenv

load_dotenv()

def post_to_note(title: str, body: str) -> bool:
    email = os.getenv("NOTE_EMAIL")
    pw = os.getenv("NOTE_PASSWORD")
    if not (email and pw):
        return False

    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        page = browser.new_page()

        page.goto("https://note.com/login", wait_until="networkidle")
        page.fill('input[name="email"]', email)
        page.fill('input[name="password"]', pw)
        page.click('button[type="submit"]')
        page.wait_for_url("https://note.com/*", timeout=10_000)

        page.goto("https://note.com/notes/new", wait_until="networkidle")
        page.fill('.title-input', title)
        page.keyboard.press("Tab")
        page.keyboard.type(body)
        page.click('button:has-text("公開")')
        page.wait_for_selector('.publish-modal', timeout=5_000)
        page.click('button:has-text("公開する")')
        browser.close()
    return True
```

## .env未設定でスキップする安全設計

各posterは対応する環境変数が空の場合に `False` を返す。orchestratorは返値だけで判断するため、チャンネルを追加・削除するときは `.env` の1行を変えるだけでよい。

```bash
# .env.template
ZENN_GITHUB_PAT=        # 空→zenn skip
ZENN_REPO=username/zenn-content
QIITA_TOKEN=            # 空→qiita skip
QIITA_MIN_INTERVAL_HOURS=5
DEVTO_API_KEY=          # 空→devto skip
BLUESKY_HANDLE=         # 空→bluesky skip
BLUESKY_PASSWORD=
NOTE_EMAIL=             # 空→note skip
NOTE_PASSWORD=
```

```python
# orchestrator/run.py（poster呼び出し部分）
from posters import zenn, qiita, devto, bluesky, note_cdp

DISPATCH = [
    ("zenn",     lambda: zenn.push_to_zenn(article_path)),
    ("qiita",    lambda: qiita.post_to_qiita(title, body, tags)),
    ("devto",    lambda: devto.post_to_devto(title, body, tags)),
    ("bluesky",  lambda: bluesky.post_to_bluesky(summary)),
    ("note",     lambda: note_cdp.post_to_note(title, body)),
]

results = {}
for name, fn in DISPATCH:
    try:
        results[name] = fn()
    except Exception as e:
        results[name] = False
        print(f"[{name}] ERROR: {e}")

print({k: "ok" if v else "skip/fail" for k, v in results.items()})
```

この設計により、**5チャンネル中どれが未設定でもpipelineは止まらず**、設定済みチャンネルだけが実行される。Bluesky凍結（X凍結と同様のリスク）が発生しても `.env` の該当行を空にすれば即座に切り離せる。
