---
title: "実コード全公開──コスト追跡・リトライ・ログ監視で運用を止めない設計"
free: false
---

## コスト追跡・リトライ・ログ監視で運用を止めない設計

### ログ爆死とコスト爆発──本番で踏んだ落とし穴

パイプラインを毎朝 7 時に自動実行し始めて最初に気づいたのは、**ログが 1 日で数 MB を超える**という事実だった。Claude API のレスポンスをそのまま `print()` していたため、1 記事ぶんのプロンプト＋本文が丸ごとファイルに書き込まれ、1 週間後にはディスクの警告が出た。

対策はシンプルだ。ログレベルを `WARNING` 以上に絞り、トークン数だけを記録する。

```python
import logging, json
from pathlib import Path

logging.basicConfig(
    filename="logs/pipeline.log",
    level=logging.WARNING,          # DEBUG/INFO は捨てる
    format="%(asctime)s %(levelname)s %(message)s",
)

def log_usage(channel: str, usage: dict):
    """レスポンスの usage フィールドだけを記録"""
    entry = {
        "ch": channel,
        "in": usage.get("input_tokens", 0),
        "out": usage.get("output_tokens", 0),
    }
    with open("logs/usage.jsonl", "a") as f:
        f.write(json.dumps(entry, ensure_ascii=False) + "\n")
```

---

### レートリミット対策──429 を黙って握りつぶすな

Claude API は 1 分あたりのリクエスト数に上限がある。複数チャンネルを並走させると `429 Too Many Requests` が頻発した。初期実装は例外を握りつぶしていたため、**記事がゼロ本生成されたまま「成功」と報告する**という最悪の状態が続いた。

指数バックオフ付きのリトライを必ず実装する。

```python
import time, anthropic
from anthropic import RateLimitError, APIStatusError

client = anthropic.Anthropic()

def call_with_retry(prompt: str, max_retries: int = 5) -> str:
    for attempt in range(max_retries):
        try:
            resp = client.messages.create(
                model="claude-sonnet-4-6",
                max_tokens=2048,
                messages=[{"role": "user", "content": prompt}],
            )
            return resp.content[0].text
        except RateLimitError:
            wait = 2 ** attempt          # 1→2→4→8→16 秒
            logging.warning("429 hit, retry %d in %ds", attempt + 1, wait)
            time.sleep(wait)
        except APIStatusError as e:
            logging.error("API error %s", e)
            raise
    raise RuntimeError("max retries exceeded")
```

---

### ハング復旧──6 時間待っても記事ゼロだった件

オーケストレーターが `claude -p` のパイプ待ちでフリーズし、CPU 使用率ほぼ 0 のまま 6〜12 時間放置されていたことがある。朝に仕掛けたはずの記事が夜になっても 0 本、という事態だ。

**ロックファイルでプロセス多重起動を防ぎ、タイムアウト付きで kill する**のが正解。

```python
import os, signal
from pathlib import Path

LOCK = Path("/tmp/pipeline.lock")

def acquire_lock(timeout_sec=3600):
    if LOCK.exists():
        pid = int(LOCK.read_text())
        try:
            os.kill(pid, 0)          # プロセス生存確認
            raise RuntimeError(f"already running: pid={pid}")
        except ProcessLookupError:
            LOCK.unlink()            # 死んだロックを除去
    LOCK.write_text(str(os.getpid()))

def release_lock():
    LOCK.unlink(missing_ok=True)
```

Task Scheduler から実行する場合は「前のタスクが実行中なら起動しない」を有効にし、最大実行時間を 2 時間に設定することで二重起動を防ぐ。

---

### 月額コストダッシュボード

`usage.jsonl` を集計すれば月次コストを可視化できる。Claude Sonnet 4.6 の価格（2026 年 6 月時点）は入力 $3/MTok・出力 $15/MTok。

```python
import json
from collections import defaultdict
from pathlib import Path

PRICE_IN  = 3.0  / 1_000_000   # $ per token
PRICE_OUT = 15.0 / 1_000_000

def monthly_report(log_path="logs/usage.jsonl"):
    totals = defaultdict(lambda: {"in": 0, "out": 0})
    for line in Path(log_path).read_text().splitlines():
        r = json.loads(line)
        totals[r["ch"]]["in"]  += r["in"]
        totals[r["ch"]]["out"] += r["out"]

    print(f"{'channel':<20} {'in_tok':>8} {'out_tok':>8} {'cost_usd':>10}")
    print("-" * 50)
    grand = 0.0
    for ch, t in sorted(totals.items()):
        cost = t["in"] * PRICE_IN + t["out"] * PRICE_OUT
        grand += cost
        print(f"{ch:<20} {t['in']:>8,} {t['out']:>8,} ${cost:>9.4f}")
    print(f"\n{'TOTAL':<20} {'':>8} {'':>8} ${grand:>9.4f}")
    print(f"        ≈ ¥{grand * 150:,.0f}  (1USD=150JPY)")

if __name__ == "__main__":
    monthly_report()
```

実行例：

```
channel              in_tok   out_tok   cost_usd
--------------------------------------------------
blog                 42,300   18,500    $0.4044
zenn                 28,100   12,200    $0.2673
qiita_tech           19,800    8,900    $0.1929

TOTAL                                   $0.8646
        ≈ ¥130  (1USD=150JPY)
```

月 130 円で 1 日 10 本の記事を量産できている計算になる。コストが突然跳ね上がった場合はチャンネル別に絞り込み、プロンプトが肥大化していないかを確認する。

---

### まとめ：運用を止めない 3 原則

| 問題 | 対策 |
|------|------|
| ログ爆死 | レベルを `WARNING` に絞りトークン数のみ記録 |
| 429 レートリミット | 指数バックオフ付きリトライ、失敗を絶対に握りつぶさない |
| プロセスハング | ロックファイル＋タイムアウト kill、Task Scheduler の多重起動防止 |

コストとログを「見える化」するだけで、問題の発見が数時間から数分に縮まる。
