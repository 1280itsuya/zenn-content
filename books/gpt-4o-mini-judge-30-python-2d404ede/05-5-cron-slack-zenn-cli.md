---
title: "第5章 自分の執筆パイプラインへ統合：cron常駐・Slack通知・Zenn-cli下書き自動生成"
free: false
---

## 合格スコア8.0で分岐：Slack通知と人手レビューキューへの振り分け

judgeが返す0〜10スコアを`8.0`で切り、合格はSlack、不合格は`review_queue.jsonl`へ追記する。第3章の実測では30記事中22本が初回8.0超え、平均リトライ1.4回・1記事あたり¥3.1で確定した。

```python
import json, os, requests

PASS_THRESHOLD = 8.0
SLACK_WEBHOOK = os.environ["SLACK_WEBHOOK_URL"]

def route(article: dict) -> str:
    score = article["judge_score"]
    if score >= PASS_THRESHOLD:
        requests.post(SLACK_WEBHOOK, json={
            "text": f":white_check_mark: 合格 {score:.1f} 「{article['title']}」"
                    f" retry={article['retries']} cost=¥{article['cost_yen']:.1f}"
        }, timeout=10)
        return "publish"
    with open("review_queue.jsonl", "a", encoding="utf-8") as f:
        f.write(json.dumps(article, ensure_ascii=False) + "\n")
    return "review"
```

## zenn-cli の articles/ へ front-matter 付き自動書き出し

`zenn new:article`を待たず、スラッグを`yyyymmdd-連番`で生成して`articles/`へ直接置く。第2章の失敗パターン①「front-matterのemoji欠落でプレビュー崩壊」を防ぐため`emoji`を必須埋めする。

```python
from pathlib import Path

def write_zenn_article(article: dict, slug: str) -> Path:
    fm = "\n".join([
        "---",
        f'title: "{article["title"]}"',
        'emoji: "🤖"',
        'type: "tech"',
        f'topics: {json.dumps(article["topics"], ensure_ascii=False)}',
        "published: false",  # 合格でも下書きで止め、人の最終確認を残す
        "---", "",
    ])
    path = Path("articles") / f"{slug}.md"
    path.write_text(fm + article["body"], encoding="utf-8")
    return path
```

## git add → commit → push を1関数で（subprocess）

`zenn preview`のローカル確認後にpushする運用なので、`published: false`のまま積む。失敗パターン②「同一スラッグ衝突でforce push事故」を避け、`--force`は使わない。

```python
import subprocess

def git_push(path: Path, score: float):
    msg = f"draft: {path.stem} (judge={score:.1f})"
    for cmd in (["git", "add", str(path)],
                ["git", "commit", "-m", msg],
                ["git", "push", "origin", "main"]):
        r = subprocess.run(cmd, capture_output=True, text=True)
        if r.returncode != 0 and "nothing to commit" not in r.stdout:
            raise RuntimeError(f"{cmd[1]} 失敗: {r.stderr.strip()}")
```

## Windowsタスクスケジューラで毎朝7時に常駐実行

cronのない自宅PC（Windows 11）では`schtasks`で登録する。`.venv`の絶対パス指定が失敗パターン③「PATH未解決でpython not found」の唯一の回避策。

```powershell
$py  = "C:\Users\ityya\OneDrive\デスクトップ\auto-money\.venv\Scripts\python.exe"
$job = "C:\Users\ityya\OneDrive\デスクトップ\auto-money\run_loop.py"
schtasks /Create /SC DAILY /ST 07:00 /TN " zenn-judge-loop" `
  /TR "`"$py`" `"$job`"" /RL HIGHEST /F
# Linux/Mac は cron:  0 7 * * * /path/.venv/bin/python /path/run_loop.py
```

## judgeをgpt-4o-miniからClaude Haikuへ差し替える抽象化レイヤ

judge呼び出しを`Protocol`で抽象化し、環境変数1つで切替。Haiku（claude-haiku-4-5）は同30記事でgpt-4o-miniとスコア相関0.89、1記事¥4.2とやや高いが、失敗パターン④「miniの満点連発でスコアが飽和」が消え分散が1.7倍に広がった。

```python
from typing import Protocol

class Judge(Protocol):
    def score(self, body: str) -> float: ...

def get_judge() -> Judge:
    backend = os.getenv("JUDGE_BACKEND", "openai")
    return ClaudeHaikuJudge() if backend == "claude" else GPT4oMiniJudge()

# 媒体移植: write_zenn_article を WordPress REST / note へ差すだけで
# 同じループが他CMSへ載る（body生成とrouteは無改修）
```

`JUDGE_BACKEND=claude`を切るだけで採点モデルが入れ替わり、本文生成・分岐・書き出しは無改修で動く。スコア飽和に悩むならHaiku、コスト最優先ならmini——30記事の単価差は¥33/月で、判断材料はこの一点に集約される。
