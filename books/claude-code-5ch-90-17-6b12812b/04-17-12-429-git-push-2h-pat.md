---
title: "失敗ログ17件：ハング12回・429連発・git push 2hブロック・PAT漏洩の原因コード"
free: false
---

章の目的を一文で言語化してから書きます：**90日間17件の失敗を根本原因コード・損失時間・修正差分のセットで全件公開し、読者が同一ミスをゼロに抑えられる防止パターンを手に入れる章。**

---

## 失敗ログ17件：ハング12回・429連発・git push 2hブロック・PAT漏洩の原因コード

90日間の実運用で発生した失敗17件を根本原因・損失時間・修正差分のセットで全件公開する。

---

## ハング12件：`claude -p` パイプがTTY待ちで最長6h・CPU≈0%になる仕組み

損失時間合計37h。`subprocess.run(["claude", "-p", prompt])` がTTYを要求したまま非対話環境でブロックする。CPU≈0%なのにプロセスが終了しないのがシグネチャ。

**原因コード（これが止まる）**
```python
import subprocess
result = subprocess.run(
    ["claude", "-p", prompt],
    capture_output=True, text=True,  # TTY check が通らず無限待ち
)
```

**修正コード**
```python
import os, subprocess

result = subprocess.run(
    ["claude", "--no-interactive", "-p", prompt],
    capture_output=True, text=True,
    timeout=300,
    env={**os.environ, "TERM": "dumb"},  # TTY要求を潰す
)
if result.returncode != 0:
    raise RuntimeError(f"claude failed: {result.stderr[:200]}")
```

Task Scheduler / cron 経由の起動時は `TERM=dumb` を環境変数に必ず追加する。

---

## Qiita 429連発：レートゲート未実装で1日57件投稿・アカウント警告メール到着まで

90日中3回発生、アカウント警告1回。Qiita APIの実質上限は10〜20件/日程度。初期実装にスリープがなく57件/日を投稿した翌朝に警告メールが届いた。

**原因コード**
```python
for article in articles:
    post_to_qiita(article)  # インターバルなし・上限チェックなし
```

**修正コード（5hインターバル強制）**
```python
import time, os

QIITA_MIN_INTERVAL_H = float(os.getenv("QIITA_MIN_INTERVAL_HOURS", "5"))
QIITA_DAILY_MAX      = int(os.getenv("QIITA_DAILY_MAX", "3"))

def post_with_rate_gate(articles: list[dict]) -> None:
    posted_today = get_today_post_count()  # jsonl等から当日投稿数を取得
    for article in articles:
        if posted_today >= QIITA_DAILY_MAX:
            print(f"Daily cap {QIITA_DAILY_MAX} reached. Abort.")
            break
        post_to_qiita(article)
        posted_today += 1
        time.sleep(QIITA_MIN_INTERVAL_H * 3600)
```

---

## git push 2hブロック：GitHub認証ダイアログのTTY待ちを `GIT_TERMINAL_PROMPT=0` で解消

2回発生・各1.5〜2h損失。Task Scheduler から起動した `git push origin main` が認証UIをTTYに表示しようとして永久待ち。

**原因の確認**
```bash
# PATがremote URLに未設定の場合、以下がハングする
GIT_TERMINAL_PROMPT=1 git push origin main
```

**修正：PAT動的注入 + ダイアログ無効化**
```bash
source .env  # GITHUB_PAT=ghp_XXXX が入っている
git remote set-url origin "https://${GITHUB_PAT}@github.com/yourname/zenn-content.git"
GIT_TERMINAL_PROMPT=0 git push origin main
# push後にURLをリセット（.git/configにPATを残さない）
git remote set-url origin "https://github.com/yourname/zenn-content.git"
```

Python から呼ぶ場合:
```python
env = {**os.environ, "GIT_TERMINAL_PROMPT": "0"}
subprocess.run(["git", "push", "origin", "main"], env=env, check=True, timeout=60)
```

---

## PAT平文漏洩：remote URLへの直接埋め込みを `git log` で検出・revoke する手順

1回発生・再設定コスト2h。`git remote set-url` の変更を `git commit` してしまい `.git/config` ごとPATがpushされ、GitHub自動検出で即時無効化された。

**漏洩チェックコマンド**
```bash
# remote URLにトークンが埋まっているか確認
git remote -v | grep -E 'ghp_|github_pat_'

# commit履歴にトークンが紛れていないか確認
git log --all --oneline -p | grep -E 'ghp_[A-Za-z0-9]{36}'
```

**安全な管理パターン：.env から動的注入・push後リセット**
```bash
# .gitignore に .env を必ず追加済みであること
echo ".env" >> .gitignore

# .env の中身
# GITHUB_PAT=ghp_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

source .env
git remote set-url origin "https://${GITHUB_PAT}@github.com/yourname/zenn-content.git"
git push origin main
git remote set-url origin "https://github.com/yourname/zenn-content.git"  # 即リセット
```

---

## 品質ドリフト：タイトル≠本文乖離で没率70%になったエージェント間テーマ伝達の欠落

継続的に発生。タイトル生成・本文生成・投稿の3プロセスが分離しており、タイトル決定時のテーマが本文エージェントに渡らないまま独立生成されていた。「Claude API活用術」タイトルに「ChatGPT副業」本文が紐づく乖離が多発し、人手レビュー没率70%に到達。

**原因：エージェント間テーマ伝達なし**
```python
title = title_agent.generate(topic)   # ここでテーマが確定する
body  = body_agent.generate(topic)    # titleを参照しない独立生成
post(title=title, body=body)          # 乖離が埋め込まれた状態で投稿
```

**修正：タイトルを本文エージェントに注入 + キーワード一致検証**
```python
def generate_article(topic: str) -> dict:
    title = title_agent.generate(topic)

    body = body_agent.generate(
        topic=topic,
        title_constraint=f"タイトル『{title}』のテーマに厳密に沿うこと。他テーマの内容を混入しない。"
    )

    # タイトル主要語が本文に含まれるか検証
    title_kws = [w for w in title.split("・") if len(w) > 1]
    missing = [kw for kw in title_kws if kw not in body]
    if missing:
        raise ValueError(f"Title-body drift: missing {missing} — abort before post")

    return {"title": title, "body": body}
```

`ValueError` で止めて人手確認に回すことで没投稿を0に抑えた。自動投稿ループ内でこの例外をキャッチせずに伝播させるのがポイント——握り潰すと再び70%没率に戻る。
