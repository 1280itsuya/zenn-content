---
title: "第5章　Claude APIパイプライン運用でハマった落とし穴と対処法"
free: false
---

## 第5章　Claude APIパイプライン運用でハマった落とし穴と対処法

自動化パイプラインは「動いた」瞬間が終わりではない。本番運用を重ねるほど、設計時には想定しなかったバグが積み重なる。この章では筆者が実際に踏んだ落とし穴を、原因・症状・修正コードとともに列挙する。

---

## 落とし穴① `.env` の読み込み忘れで投稿が静かにスキップされる

**症状**: スクリプトはエラーなく完走するが、Zennへの投稿が一件も行われない。ログには `ZENN_PAT not set – skipping` の一行だけ。

**原因**: `os.environ` は実行時の環境変数しか参照しない。`python-dotenv` をインポートしても `load_dotenv()` を呼ばないと `.env` が読まれない。

```python
# NG: インポートだけして呼び出し忘れ
from dotenv import load_dotenv
import os

token = os.getenv("ZENN_PAT")  # → None

# OK: ファイル冒頭で必ず呼ぶ
from dotenv import load_dotenv
load_dotenv()
import os

token = os.getenv("ZENN_PAT")  # → 正しく読み込まれる
```

モジュールが複数ファイルに分かれている場合は、エントリポイントとなる `orchestrator.py` の先頭で一度だけ呼べばよい。

---

## 落とし穴② Qiita/外部APIの429エラーでパイプライン全体が止まる

**症状**: 朝の一括投稿で Qiita が `429 Too Many Requests` を返し、それ以降のタスクも連鎖停止する。

**原因**: 複数チャンネルが同時にAPIを叩き、プラットフォーム側のレートリミットに衝突する。

**修正**: 投稿側に最小間隔ゲートを設ける。

```python
import time
from pathlib import Path

QIITA_LAST_POST_FILE = Path(".qiita_last_post")
MIN_INTERVAL_HOURS = float(os.getenv("QIITA_MIN_INTERVAL_HOURS", "5"))

def can_post_qiita() -> bool:
    if not QIITA_LAST_POST_FILE.exists():
        return True
    elapsed = time.time() - QIITA_LAST_POST_FILE.stat().st_mtime
    return elapsed >= MIN_INTERVAL_HOURS * 3600

def post_to_qiita(body: str):
    if not can_post_qiita():
        print("Rate gate: skipping Qiita post")
        return
    # ... 投稿処理 ...
    QIITA_LAST_POST_FILE.touch()
```

環境変数 `QIITA_MIN_INTERVAL_HOURS` で間隔を調整できるようにしておくと、実運用で柔軟に対応できる。

---

## 落とし穴③ タイトルと本文でテーマが乖離する「品質ドリフト」

**症状**: タイトルは「Pythonで家計簿を自動化する方法」なのに、本文はAI倫理の話になっている。Claude APIへの連続リクエスト中にコンテキストがドリフトする。

**原因**: プロンプトにテーマ固定の制約がなく、モデルが前後のコンテキストを引きずる。

**修正**: 生成後に一致検証を挟み、不一致なら中止する。

```python
def verify_content_integrity(title: str, body: str, theme: str) -> bool:
    check_prompt = f"""
タイトル: {title}
テーマ: {theme}
本文の最初の200字: {body[:200]}

タイトルと本文が同じテーマを扱っているか? "YES" か "NO" だけ答えよ。
"""
    result = claude_client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=10,
        messages=[{"role": "user", "content": check_prompt}]
    ).content[0].text.strip()

    return result.upper() == "YES"

# 使用例
if not verify_content_integrity(title, body, theme):
    raise ValueError(f"Content drift detected: '{title}' does not match theme '{theme}'")
```

検証にはコストの低い `claude-haiku` を使うことで、品質ゲートのコストを抑えられる。

---

## 落とし穴④ git push が TTY 待ちでハングする

**症状**: Zenn への自動投稿（git push）が深夜に止まり、翌朝まで約2時間プロセスが生き続ける。CPUはほぼ0%。

**原因**: GitHub の認証ダイアログがターミナル入力（TTY）を要求するが、自動化環境にはTTYがないため無限待機になる。

**修正**: PAT（Personal Access Token）をリモートURLに埋め込み、非対話化する。

```python
import subprocess, os

def push_to_zenn_repo(repo_path: str):
    pat = os.getenv("GITHUB_PAT")
    remote = f"https://{pat}@github.com/your-account/zenn-content.git"

    env = os.environ.copy()
    env["GIT_TERMINAL_PROMPT"] = "0"  # 認証ダイアログを完全に無効化

    subprocess.run(
        ["git", "push", remote, "main"],
        cwd=repo_path,
        env=env,
        timeout=60,
        check=True
    )
```

`GIT_TERMINAL_PROMPT=0` を設定することで、認証が必要な場面では待機ではなくエラーで即座に終了するようになり、ハングを防げる。なお PAT はリモートURLに平文で入るため、`.env` に格納してリポジトリにはコミットしないこと。

---

## 落とし穴⑤ オーケストレーターが `claude -p` パイプ待ちで長時間ハングする

**症状**: パイプラインを手動起動すると途中で止まり、6〜12時間後に見ると何も投稿されていない。

**原因**: `claude -p`（Claude CLI）へのパイプが応答を待ち続けるが、並列タスクがロックファイルを握ったまま終了しないケースがある。

**修正**: タイムアウトとロック除去を組み合わせる。

```python
import subprocess, os
from pathlib import Path

LOCK_FILE = Path("/tmp/pipeline.lock")

def run_with_timeout(cmd: list, timeout_sec: int = 300):
    try:
        return subprocess.run(cmd, timeout=timeout_sec, check=True, capture_output=True, text=True)
    except subprocess.TimeoutExpired:
        print(f"Timeout after {timeout_sec}s: {' '.join(cmd)}")
        if LOCK_FILE.exists():
            LOCK_FILE.unlink()
        raise
```

タスクスケジューラ（Windowsなら「タスクスケジューラ」）から起動する場合は、前回プロセスが残っていないかを確認するラッパーをかぶせると多重起動も防げる。

---

## まとめ: 運用バグに共通するパターン

| バグの種類 | 共通原因 | 対策の型 |
|---|---|---|
| `.env` 読み忘れ | 初期化の漏れ | エントリポイントで一括呼び出し |
| 429 レートリミット | 同時投稿の集中 | ファイルベースのゲート |
| 品質ドリフト | プロンプト制約不足 | 生成後の検証エージェント |
| git push ハング | TTY 依存の認証 | `GIT_TERMINAL_PROMPT=0` + PAT |
| CLI パイプハング | ロック競合 | タイムアウト + ロック除去 |

自動化の鉄則は「失敗を黙って飲み込まないこと」だ。スキップログは必ず出力し、タイムアウトは必ず設定する。それだけで運用バグの8割は翌朝までに発見できる。
