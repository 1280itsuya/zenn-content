---
title: "オーケストレーター実装：LLM-as-Judge品質ゲートとagent-loop設計コード全公開"
free: false
---

## claude -p 非対話起動：subprocess.runラッパーとタイムアウト60秒の壁

`claude -p` をターミナルから叩くと、GitHub認証ダイアログが出た瞬間にTTY待ちで無限ハングする（障害ログ#3/#7/#11、実測2〜12h停止）。`GIT_TERMINAL_PROMPT=0` と `timeout=60` を同時に設定して封じる。

```python
# src/run_agent.py
import subprocess
import os

def run_claude(prompt: str, timeout: int = 60) -> str:
    env = {**os.environ, "GIT_TERMINAL_PROMPT": "0", "CLAUDE_NO_TTY": "1"}
    result = subprocess.run(
        ["claude", "-p", prompt],
        capture_output=True,
        text=True,
        timeout=timeout,
        env=env,
    )
    if result.returncode != 0:
        raise RuntimeError(f"claude exit {result.returncode}: {result.stderr[:300]}")
    return result.stdout.strip()
```

`timeout=60` を外すと6〜12h単位でハングした。短すぎると長文生成が途中切れするため60秒が実測の下限値。

## agents.json Configure-Driven設計：17エージェントを1ファイルで集中管理

エージェント設定をソースに埋め込むと、スケジュールを1件変えるたびにデプロイが走る。`config/agents.json` に全設定を外出しして、オーケストレーターは読むだけにした。

```json
{
  "agents": [
    {
      "name": "qiita_tech",
      "enabled": true,
      "schedule": "07:00",
      "quality_threshold": 7,
      "max_retries": 3,
      "module": "src.channels.qiita_tech"
    },
    {
      "name": "zenn_book",
      "enabled": true,
      "schedule": "07:15",
      "quality_threshold": 8,
      "max_retries": 2,
      "module": "src.channels.zenn_book"
    }
  ]
}
```

`"enabled": false` にするだけで無効化できる。90日間で計11回の一時停止を、コード変更なしでこのフラグだけで処理した。

## LLM-as-Judge：スコア0〜10の判定プロンプト設計と実測23%却下率

Judgeプロンプトは採点基準を数値化し、出力をJSONのみに強制する。自由記述を許すとパースエラーが頻発した（実測エラー率18% → JSON強制後0.4%）。

```python
# src/quality_judge.py
import json

JUDGE_PROMPT = """\
記事を0〜10点で採点せよ。出力はJSONのみ。

採点基準:
- 具体的なコードまたは数値を含む      +3
- タイトルと本文の主題が一致する      +3
- 検索意図に正面から答えている        +2
- AI常套句（「思います」等）がない    +2

記事（先頭2000字）:
{content}

出力: {{"score": <0-10の整数>, "reason": "<50字以内>"}}
"""

def judge_quality(content: str, run_fn=None) -> dict:
    if run_fn is None:
        from .run_agent import run_claude as run_fn
    raw = run_fn(JUDGE_PROMPT.format(content=content[:2000]), timeout=30)
    return json.loads(raw)
```

90日間で合計1,247件を判定し、287件（23.0%）がスコア7未満で却下。却下の主因はタイトル≠本文ドリフト（57%）とコードブロック欠如（31%）。

## スコア7未満リトライループ：max_retries=3と¥0コストの根拠

再生成は最大3回に固定する。4回目以降は合格確率が1.2%以下に収束し、Maxプラン以外ではコストだけ嵩む（1件あたり約¥2〜5）。

```python
# src/orchestrator.py（リトライ部分）
from .run_agent import run_claude
from .quality_judge import judge_quality

def generate_with_retry(agent_cfg: dict) -> str | None:
    threshold = agent_cfg.get("quality_threshold", 7)
    max_retries = agent_cfg.get("max_retries", 3)

    for attempt in range(max_retries):
        try:
            content = run_claude(agent_cfg["prompt"], timeout=60)
            verdict = judge_quality(content)
            if verdict["score"] >= threshold:
                return content
            print(f"[REJECT] attempt={attempt+1} score={verdict['score']}: {verdict['reason']}")
        except Exception as e:
            print(f"[ERROR] attempt={attempt+1}: {e}")

    print(f"[SKIP] {agent_cfg['name']} — 全{max_retries}回却下")
    return None
```

Claude Max定額プランでは再生成コストが¥0のため、リトライを惜しまずにかけられる構造になっている。

## fcntlによるlockファイル排他制御：並列実行で破損ゼロ件を維持した実装

複数エージェントが同時に `data/latest.json` へ書き込むと、後から完了した側が前の結果を上書きする。`fcntl.flock` で排他ロックを取得し、書き込み完了後に解放する。

```python
# src/safe_writer.py
import fcntl
import json
from pathlib import Path

def safe_write_json(path: str, data: dict) -> None:
    lock_path = path + ".lock"
    with open(lock_path, "w") as lf:
        # LOCK_NB を外すと他プロセスはロック解放まで待機する（即エラーにしない）
        fcntl.flock(lf, fcntl.LOCK_EX)
        try:
            Path(path).write_text(
                json.dumps(data, ensure_ascii=False, indent=2),
                encoding="utf-8",
            )
        finally:
            fcntl.flock(lf, fcntl.LOCK_UN)
```

Windows環境では `fcntl` が存在しないため `msvcrt.locking` に差し替える（付録B参照）。この実装で90日間・並列最大8プロセス運用を通してデータ破損はゼロ件。
