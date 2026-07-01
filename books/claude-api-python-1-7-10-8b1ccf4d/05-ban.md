---
title: "落とし穴と対策——レート制限・タイトル本文ドリフト・プラットフォームBANの実体験"
free: false
---

## 落とし穴と対策——レート制限・タイトル本文ドリフト・プラットフォームBANの実体験

自動投稿パイプラインは「動いた」瞬間が最も油断しやすい。筆者が実際にぶつかった3つの失敗と、それぞれのコード修正を順番に解説する。

---

## 失敗①　Qiita 429——9本のスクリプトが同時バースト

### 何が起きたか

ジャンルごとに独立した生成スクリプトを9本動かしていた。全部が「毎朝7時」にスケジュールされていたため、Qiita APIへのPOSTが7:00に集中。翌朝に確認すると投稿数0本、レスポンスは `HTTP 429 Too Many Requests` だった。

### 根本原因

スクリプトごとに `posters/qiita.py` を個別呼び出ししていたが、「最後にいつ投稿したか」を誰も共有していなかった。

### 修正：集中レートゲート

`post_item` に「最終投稿時刻をファイルで共有し、N時間未満ならスキップ」するロジックを加える。

```python
import json, time
from pathlib import Path

_STAMP = Path("data/_qiita_last_post.json")
MIN_INTERVAL = int(os.getenv("QIITA_MIN_INTERVAL_HOURS", "5")) * 3600

def _is_rate_gated() -> tuple[bool, int]:
    if not _STAMP.exists():
        return False, 0
    last = json.loads(_STAMP.read_text())["ts"]
    wait = int(MIN_INTERVAL - (time.time() - last))
    return wait > 0, max(wait, 0)

def post_item(title: str, body: str) -> dict:
    gated, wait_sec = _is_rate_gated()
    if gated:
        return {"posted": False, "skip": "rate_gate",
                "next_in_min": wait_sec // 60}
    # ... 実際のPOST処理 ...
    _STAMP.write_text(json.dumps({"ts": time.time()}))
    return {"posted": True}
```

`QIITA_MIN_INTERVAL_HOURS=5` に設定すると、7時・13時・19時（6時間間隔）のスケジュールで1日最大3本が確実に通る。サーバが429を返した場合も `_STAMP` を更新してクールダウンを記録する。

---

## 失敗②　Zenn git push——2時間ハングで1冊が闇に消えた

### 何が起きたか

Zennはリポジトリ連携で公開する仕組みのため、記事生成後に `git push` が必要になる。夜間の無人ループで push が止まり、2時間後に確認するとターミナルに以下のメッセージが残っていた。

```
could not read Username for 'https://github.com'
User cancelled dialog
/dev/tty: No such device
```

ヘッドレス実行ではGUI認証ダイアログを表示するデバイスが存在せず、gitが無限に待ち続けていた。

### 修正：PAT埋め込み＋GCMダイアログ封殺

```python
import subprocess, os

def _push(repo_dir: str) -> None:
    token = os.environ["GITHUB_TOKEN"]
    user  = os.environ["GITHUB_USERNAME"]
    # origin に PAT を一時的に埋め込んだURLで push（永続保存しない）
    remote = f"https://{user}:{token}@github.com/{user}/zenn-content.git"
    env = {
        **os.environ,
        "GIT_TERMINAL_PROMPT": "0",   # TTYプロンプトを禁止
        "GCM_INTERACTIVE": "never",   # Git Credential Manager GUIを封殺
    }
    subprocess.run(
        ["git", "-c", "credential.helper=", "push", remote, "main"],
        cwd=repo_dir, env=env, check=True, timeout=60
    )
```

ポイントは3つある。①`GIT_TERMINAL_PROMPT=0` でTTYへの問い合わせを完全に禁止し、失敗なら即エラーにする。②`GCM_INTERACTIVE=never` でWindowsのCredential Managerウィンドウを抑止する。③PAT埋め込みURLはプロセス引数内にのみ存在し、`~/.gitconfig` や `.git/config` には書き込まない。

なお push 前に `git fetch && git rebase FETCH_HEAD` でリモートの先行コミットを取り込んでおかないと non-fast-forward で弾かれる。衝突した場合は `rebase --abort` してローカル保存に留め、次サイクルで再試行する設計にする。

---

## 失敗③　タイトル本文ドリフト——スパム判定と読者離脱の二重罰

### 何が起きたか

生成スクリプトを量産する過程で、タイトルと本文の主題がずれた記事が大量発生した。典型例：

- タイトル：「楽天モバイル vs LINEMO 徹底比較」
- 本文：「楽天カードと三井住友カードのポイント還元率を比べると……」

LLMの品質ゲートは文章の流暢さを9点以上でPASSするが、**タイトルと本文の一致は見ていない**。Qiitaはこの種の記事をテンプレスパムと判定してviewが伸びず、投稿制限の対象にもなった。

### 原因

プロンプト内の「例示」や「当日優先トピック」変数にモデルが引っ張られ、冒頭で固定したはずのテーマから逸脱する。

### 修正：テーマ固定＋本文一致検証

```python
def _on_theme(theme: str, body: str, threshold: float = 0.30) -> bool:
    """テーマの主要語が本文に一定割合含まれるか検証する"""
    from collections import Counter
    import re
    # 2文字以上の語を2-gramで分割してキーワード集合を作る
    words = set(re.findall(r'[\w\u3040-\u9fff]{2,}', theme))
    if not words:
        return True
    hits = sum(1 for w in words if w in body)
    return hits / len(words) >= threshold

class BlogAgent:
    def write(self, theme: str) -> str:
        prompt = f"必ず「{theme}」だけを主題として書くこと。他のトピックに触れてはならない。\n\n{theme}について……"
        for attempt in range(2):
            body = self._call_llm(prompt)
            if _on_theme(theme, body):
                return body
            # テーマ逸脱 → 再生成
        raise ValueError(f"テーマ逸脱が解消されませんでした: {theme}")
```

比較記事では両アイテムが本文に登場するかを追加検証し、欠落したらエラーを上げて**公開しない**。「崩れた記事を出さない」を投稿件数より優先することで、プラットフォームからの信頼を守る。

---

## まとめ：3つの失敗に共通する教訓

| 問題 | 原因 | 対策のキー |
|------|------|-----------|
| Qiita 429 | スクリプト間で状態を共有していない | ファイルで最終投稿時刻を共有するレートゲート |
| Zenn ハング | ヘッドレス環境でGUIダイアログ待ち | `GIT_TERMINAL_PROMPT=0` + PAT埋め込みURL |
| テーマドリフト | LLM品質ゲートが主題一致を見ていない | プロンプト冒頭固定＋本文検証→不一致は中止 |

自動化のバグは「動いていない」ではなく「動いているのに誰も知らないまま壊れている」形で現れる。各スクリプトに返却値でスキップ理由を記録し、朝のダッシュボードで `"posted": False` が並んでいないかを必ず確認する習慣をつけると、障害を数時間以内に発見できる。
