---
title: "第3章 GitHub APIにインラインコメントを返す：reviewComments投稿の落とし穴4つ"
free: false
---

## 422エラーの正体：diff_hunk外のlineは弾かれる

`gh api .../pulls/{n}/comments` に `path` と `line` を渡しても、その行が差分（diff_hunk）に含まれていなければ GitHub は `422 Unprocessable Entity` を返す。実運用3週間で投稿失敗127件のうち89件がこれだった。Claudeは変更行の前後（コンテキスト行）まで指摘しがちで、ここに弾かれる。

```python
import subprocess, json

def diff_lines(owner, repo, n, path):
    out = subprocess.run(
        ["gh","api",f"repos/{owner}/{repo}/pulls/{n}/files","--paginate"],
        capture_output=True, text=True).stdout
    for f in json.loads(out):
        if f["filename"] != path: continue
        # patch内の '+' 行番号だけ抽出（コメント可能な行）
        ln, valid = 0, set()
        for row in f["patch"].splitlines():
            if row.startswith("@@"):
                ln = int(row.split("+")[1].split(",")[0]); continue
            if row.startswith("+"): valid.add(ln); ln += 1
            elif not row.startswith("-"): ln += 1
        return valid
    return set()
```

投稿前に `line in diff_lines(...)` を必ず検証し、外れる指摘はファイル全体コメントへ降格させると422が89→3件に減った。

## position廃止後はside/lineで右側差分を指定する

旧 `position`（diff先頭からの相対行）は2023年以降非推奨で、現行は `side` + `line` 指定。変更後の行を指すなら `side: "RIGHT"`、削除行を指すなら `LEFT`。混同すると別行に貼られレビュー漏れに見える。

```bash
gh api repos/$OWNER/$REPO/pulls/$PR/comments \
  -f commit_id="$SHA" \
  -f path="src/auth.py" \
  -F line=42 \
  -f side="RIGHT" \
  -f body="line 42: トークン未失効。`exp` 検証を追加"
```

`commit_id` はPRの最新HEAD SHA（`gh pr view $PR --json headRefOid -q .headRefOid`）を渡す。古いSHAだと過去コミットに貼られ画面に出ない。

## fingerprint方式で同一指摘の再投稿を防ぐ

CIが回るたびに同じ指摘を再投稿すると、3週間でノイズ重複が412件発生した。`path:line:body先頭60字` のSHA1を指紋にし、既存コメントと突き合わせて抑止する。

```python
import hashlib
def fp(c): 
    key=f'{c["path"]}:{c["line"]}:{c["body"][:60]}'
    return hashlib.sha1(key.encode()).hexdigest()

existing = {fp(c) for c in json.loads(subprocess.run(
    ["gh","api",f"repos/{OWNER}/{REPO}/pulls/{PR}/comments","--paginate"],
    capture_output=True,text=True).stdout)}

new = [c for c in claude_comments if fp(c) not in existing]
```

これで重複が412→0件、PR画面の可読性が回復した。

## pulls/reviewsで1レビューに束ねレート制限を回避

コメントを1件ずつPOSTすると、20指摘で20リクエスト消費し `403 rate limit` に触れた。`pulls/{n}/reviews` に `comments` 配列をまとめると1リクエストで完結し、消費が20→1に減る。

```python
payload = {"commit_id": SHA, "event": "COMMENT",
           "body": f"Claude自動レビュー: {len(new)}件",
           "comments": [{"path":c["path"],"line":c["line"],
                         "side":"RIGHT","body":c["body"]} for c in new]}
subprocess.run(["gh","api",f"repos/{OWNER}/{REPO}/pulls/{PR}/reviews",
                "--input","-"], input=json.dumps(payload), text=True)
```

1件ずつ貼るのは「即時性が要る重大指摘のみ」、通常は `reviews` でバッチ送信、と使い分ければ5000req/hの枠に対し1PRあたり実消費を平均1.4リクエストへ抑えられる。
