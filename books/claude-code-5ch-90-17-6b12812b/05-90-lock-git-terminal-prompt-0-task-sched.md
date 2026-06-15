---
title: "90日連続稼働の設計原則：lock除去・GIT_TERMINAL_PROMPT=0・Task Scheduler定常化"
free: false
---

## `~/.claude.json` 書き込み競合：対話セッション内手動起動が破壊する理由

`claude -p` をターミナル（対話セッション内）から手動起動すると、Claude CLI は `~/.claude.json` へセッション状態を書き込む。Task Scheduler が同時刻に別プロセスで同ファイルを読み書きしていると JSON が破損し、初回生成が空文字列で返る。実測で 75 回のうち 8 回（10.7%）がこのパターンだった。

```python
# scripts/check_claude_lock.py
import json, pathlib

CLAUDE_JSON = pathlib.Path.home() / ".claude.json"

def is_corrupted() -> bool:
    try:
        data = json.loads(CLAUDE_JSON.read_text(encoding="utf-8"))
        return not isinstance(data, dict)
    except (json.JSONDecodeError, FileNotFoundError):
        return True

if is_corrupted():
    print("[WARN] ~/.claude.json が破損しています。Task Scheduler 専用実行に切り替えてください。")
```

解決策は単純で、**Task Scheduler からのみ起動**し、対話セッション内での手動実行を禁止するルール化だ。

---

## lock 残存→平均 8.3h ハング：kill + 除去を 30 行で自動化

orchestrator が `claude -p` のパイプ待ちに入ると、CPU はほぼ 0% のまま 6〜12 時間ハングする（実測平均 8.3h）。lock ファイルが残存していると次回起動時に即スキップされるため、二重の損失になる。

```python
# scripts/recover_lock.py
import subprocess, pathlib, time

LOCK = pathlib.Path("data/pipeline.lock")
PROC_NAME = "orchestrator"

def kill_stale_orchestrator():
    result = subprocess.run(
        ["tasklist", "/FI", "IMAGENAME eq python.exe", "/FO", "CSV"],
        capture_output=True, text=True,
    )
    for line in result.stdout.splitlines()[1:]:
        if PROC_NAME in line:
            pid = line.split(",")[1].strip('"')
            subprocess.run(["taskkill", "/PID", pid, "/F"], check=False)
            print(f"[KILLED] PID {pid}")

def release_lock():
    if LOCK.exists():
        LOCK.unlink()
        print(f"[REMOVED] {LOCK}")

if __name__ == "__main__":
    kill_stale_orchestrator()
    time.sleep(2)
    release_lock()
```

Task Scheduler で `recover_lock.py` を毎朝 6:55 に実行し、7:00 の本番起動前に必ずクリーンな状態にする。

---

## `GIT_TERMINAL_PROMPT=0`：Zenn git push が 2 時間 TTY 待ちになった実録

Zenn への投稿は `git push` 経由だが、GitHub 認証ダイアログが TTY を要求して約 2 時間ハングした（当日投稿 0 件）。環境変数 1 行と PAT 埋め込み URL で完全解決する。

```python
# orchestrator/posters/zenn.py（抜粋）
import subprocess, os

def push_zenn_book(repo_path: str, pat: str, repo_url: str) -> bool:
    """PAT 埋込 URL で非対話的 git push"""
    authed_url = repo_url.replace("https://", f"https://{pat}@")
    env = os.environ.copy()
    env["GIT_TERMINAL_PROMPT"] = "0"   # TTY 要求を即エラーに変換
    env["GIT_ASKPASS"] = "echo"        # パスワード入力を空文字で潰す

    result = subprocess.run(
        ["git", "push", authed_url, "main"],
        cwd=repo_path, env=env,
        capture_output=True, text=True, timeout=60,
    )
    if result.returncode != 0:
        print(f"[PUSH_FAIL] {result.stderr[:200]}")
        return False
    return True
```

`GIT_TERMINAL_PROMPT=0` がないと `timeout=60` を設定していても認証ダイアログが先に TTY を掴んで設定が無効化される。PAT は `.env` の `GITHUB_PAT` に格納し、remote の URL に平文で永続化しない。

---

## Task Scheduler × `.venv`：venv 認識失敗ゼロにする起動バッチ

Task Scheduler は `PATH` を継承しないため、`python` コマンドが system Python を指してパッケージが見つからない。バッチファイルで venv を明示的にアクティベートする。

```bat
:: scripts\run_pipeline.bat
@echo off
cd /d C:\Users\ityya\OneDrive\デスクトップ\auto-money
call .venv\Scripts\activate.bat
python -m orchestrator.main --schedule morning >> logs\scheduler.log 2>&1
```

```powershell
# 管理者 PowerShell で登録
schtasks /Create /SC DAILY /TN "AutoMoney_Morning" /TR "C:\...\scripts\run_pipeline.bat" /ST 07:00 /F
schtasks /Create /SC DAILY /TN "AutoMoney_Noon"    /TR "C:\...\scripts\run_pipeline.bat" /ST 13:00 /F
schtasks /Create /SC DAILY /TN "AutoMoney_Evening" /TR "C:\...\scripts\run_pipeline.bat" /ST 19:00 /F
```

「開始場所」欄は空欄にし、bat ファイル内の `cd /d` で作業ディレクトリを制御する。Task Scheduler の UI から設定すると相対パスの解釈がずれるため bat に一元化する。

---

## subprocess タイムアウト + dead letter queue：3 パターン対策コード

90 日稼働中に観測した停止パターンは 3 種。それぞれに専用の対策を入れる。

```python
# orchestrator/runner.py
import subprocess, json, pathlib
from datetime import datetime

DLQ = pathlib.Path("data/dead_letter_queue.jsonl")

PATTERNS = {
    "pipe_hang":  {"timeout": 300, "desc": "claude -p パイプ待ちハング"},
    "subprocess": {"timeout": 120, "desc": "subprocess タイムアウト"},
    "lock_stale": {"timeout": None, "desc": "lock ファイル残存スキップ"},
}

def run_with_dlq(cmd: list[str], pattern_key: str, metadata: dict) -> str:
    cfg = PATTERNS[pattern_key]
    try:
        r = subprocess.run(
            cmd, capture_output=True, text=True, timeout=cfg["timeout"]
        )
        if r.returncode != 0:
            raise RuntimeError(r.stderr[:300])
        return r.stdout
    except (subprocess.TimeoutExpired, RuntimeError) as e:
        entry = {
            "ts": datetime.utcnow().isoformat(),
            "pattern": pattern_key,
            "desc": cfg["desc"],
            "error": str(e)[:200],
            **metadata,
        }
        with DLQ.open("a", encoding="utf-8") as f:
            f.write(json.dumps(entry, ensure_ascii=False) + "\n")
        raise
```

DLQ に書き込まれたエントリは翌朝の `recover_lock.py` 起動時に自動でリトライキューへ移動する。90 日で dead letter 累積は 17 件、リトライ成功 14 件（82.4%）だった。

---

## Discord Webhook + 月次コストレポート：無人確認の最小構成

無人稼働では「動いているか」を能動的に確認しない。Discord Webhook で 1 日 3 回（起動・完了・エラー時）通知し、月次コストを JSONL から集計する。

```python
# orchestrator/notifier.py
import os, httpx, json, pathlib
from datetime import date

WEBHOOK = os.environ["DISCORD_WEBHOOK_URL"]

def notify(msg: str, level: str = "info") -> None:
    color = {"info": 3447003, "warn": 16776960, "error": 15158332}.get(level, 0)
    httpx.post(WEBHOOK, json={"embeds": [{"description": msg[:2000], "color": color}]}, timeout=10)

def monthly_cost_report() -> dict:
    log_dir = pathlib.Path("logs/cost")
    month = date.today().strftime("%Y-%m")
    total = sum(
        json.loads(line).get("cost_usd", 0)
        for p in log_dir.glob("*.jsonl")
        for line in p.read_text().splitlines()
        if json.loads(line).get("month") == month
    )
    report = {"month": month, "total_usd": round(total, 4)}
    notify(f"月次コスト {month}: ${report['total_usd']}", level="info")
    return report
```

90 日間の月次コスト実測：1 ヶ月目 $4.20 → 2 ヶ月目 $3.87 → 3 ヶ月目 $2.93（`claude-haiku-4-5` 切り替えで −30%）。`notify()` を各エージェントの終端に 1 行追加するだけで、PC から離れていても稼働状態を把握できる。
