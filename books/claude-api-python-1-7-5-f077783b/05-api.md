---
title: "ハマりどころ集――API制限・コンテンツ品質・認証エラーの実例と対処"
free: false
---

## ハマりどころ集――API制限・コンテンツ品質・認証エラーの実例と対処

自動化パイプラインは「動いた」だけでは終わらない。本番稼働して初めて露わになる罠がある。ここでは実運用で実際に踏んだ3つの落とし穴を、原因・再現条件・修正コードとともに解説する。

---

## 1. Qiita API 429エラー――レート制限を黙って突き破る

### 何が起きたか

毎朝7時に記事を5本一気に投稿するスクリプトを走らせたところ、2本目以降が無言で失敗し始めた。ログには `HTTP 429 Too Many Requests` が断続的に記録されていたが、スクリプトはエラーを無視して次の投稿へ進んでいた。結果として投稿済みカウントは正常なのに実際には1本しか公開されていない、という状態が数日続いた。

### 原因

Qiita APIは短時間に連続リクエストを送ると429を返す。問題はエラーハンドリングが `pass` で握りつぶされており、失敗を検知できていなかったこと。加えて、複数チャンネルが同じAPIキーを共用していたため、呼び出し頻度が積み重なっていた。

### 修正

投稿の最低間隔を環境変数で制御し、429を受け取ったらエクスポネンシャルバックオフで再試行するラッパーを実装した。

```python
import time, os, requests

QIITA_MIN_INTERVAL = int(os.getenv("QIITA_MIN_INTERVAL_HOURS", "5")) * 3600
_last_post_time = 0

def post_to_qiita(title: str, body: str, tags: list[str]) -> dict:
    global _last_post_time
    elapsed = time.time() - _last_post_time
    if elapsed < QIITA_MIN_INTERVAL:
        time.sleep(QIITA_MIN_INTERVAL - elapsed)

    for attempt in range(4):
        resp = requests.post(
            "https://qiita.com/api/v2/items",
            headers={"Authorization": f"Bearer {os.environ['QIITA_TOKEN']}"},
            json={"title": title, "body": body, "tags": tags, "private": False},
        )
        if resp.status_code == 429:
            wait = 2 ** attempt * 60  # 1分→2分→4分→8分
            time.sleep(wait)
            continue
        resp.raise_for_status()
        _last_post_time = time.time()
        return resp.json()
    raise RuntimeError("Qiita: 429が4回連続。本日の投稿を中断します")
```

`QIITA_MIN_INTERVAL_HOURS=5` を `.env` に追記するだけで複数チャンネルからの過剰呼び出しを防げる。

---

## 2. タイトルと本文のテーマドリフト――AIが勝手に話題を変える

### 何が起きたか

「節約術10選」というタイトルで記事を生成させると、本文の後半がいつの間にか「投資信託の始め方」に変わっていた。記事数が増えるにつれ、タイトルと本文が別テーマになっているケースが全体の2割を超えた。

### 原因

エージェントへのプロンプトでタイトルを先に確定させ、本文生成を別のAPIコールに分けていたことで、2回目の呼び出し時にタイトルが文脈から薄れ、モデルが関連トピックへ脱線した。

### 修正

本文生成プロンプトにタイトルを明示的に埋め込み、生成後に一致検証を挟む。不一致なら生成を中止してログに残す。

```python
def generate_article(title: str) -> str:
    prompt = f"""
記事タイトル: 「{title}」

このタイトルの内容だけを扱う記事本文を1200字で書いてください。
タイトルに書かれていないテーマは一切含めないこと。
"""
    body = call_claude(prompt)
    _verify_theme(title, body)
    return body

def _verify_theme(title: str, body: str) -> None:
    # タイトルのキーワードが本文に存在するか簡易チェック
    keywords = [w for w in title.replace("・", " ").split() if len(w) >= 2]
    missing = [kw for kw in keywords if kw not in body]
    if len(missing) > len(keywords) * 0.5:
        raise ValueError(
            f"テーマドリフト検出: タイトル '{title}' のキーワード {missing} が本文に不足"
        )
```

この検証を全エージェントの出口に差し込んでおくと、ドリフト記事が公開される前に止められる。

---

## 3. Git push TTYハング――Zenn投稿が2時間無音で止まる

### 何が起きたか

Task Scheduler から無人実行されるZenn投稿スクリプトが、`git push` のタイミングで毎回ハングした。手動実行では問題なく通るのに、自動実行では2時間後にタイムアウトするまで何も起きない。

### 原因

GitHubが認証情報を求めるダイアログをTTY経由で表示しようとするが、バックグラウンド実行ではTTYが存在しないため永遠に待ち続けていた。`git credential-manager` が対話モードにフォールバックしていたことが根本原因。

### 修正

リモートURLにPATを直接埋め込み、`GIT_TERMINAL_PROMPT=0` で対話を完全に無効化する。

```python
import subprocess, os

def push_zenn_repo(repo_path: str) -> None:
    pat = os.environ["GITHUB_PAT"]
    remote = f"https://{pat}@github.com/your-account/zenn-content.git"

    env = os.environ.copy()
    env["GIT_TERMINAL_PROMPT"] = "0"  # 対話プロンプトを禁止

    subprocess.run(
        ["git", "remote", "set-url", "origin", remote],
        cwd=repo_path, check=True, env=env
    )
    result = subprocess.run(
        ["git", "push", "origin", "main"],
        cwd=repo_path, capture_output=True, text=True, env=env, timeout=60
    )
    if result.returncode != 0:
        raise RuntimeError(f"git push失敗:\n{result.stderr}")
```

> **注意**: PATをURLに埋め込む方法はリモートURLがログに残るリスクがある。`git remote set-url` は都度上書きし、`.git/config` をバージョン管理に含めないこと。

---

## まとめ

| 症状 | 根本原因 | 対処の核心 |
|---|---|---|
| 投稿が無言で失敗 | 429を握りつぶすerrハンドラ | バックオフ付き再試行＋最低間隔ゲート |
| タイトル≠本文 | 別コールで文脈が希薄化 | プロンプトにタイトル明示＋出口検証 |
| git pushが2時間ハング | TTYなし環境でGit認証待ち | `GIT_TERMINAL_PROMPT=0`＋URL内PAT |

どれも「手動では動く」が「無人では壊れる」パターンだ。自動化の信頼性はエラーを黙らせないことで成り立つ。
