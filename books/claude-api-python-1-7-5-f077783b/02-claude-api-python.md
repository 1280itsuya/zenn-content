---
title: "Claude API環境構築とPythonラッパーの実装"
free: false
---

## APIキーの取得と `.env` 管理

[Anthropic Console](https://console.anthropic.com/) にログインし、**API Keys** → **Create Key** でキーを発行する。発行直後にしか全文表示されないため、即コピーすること。

`.env` ファイルにキーを書く。**絶対に Git にコミットしない。**

```
# .env
ANTHROPIC_API_KEY=sk-ant-api03-xxxxxxxxxxxx
CLAUDE_MODEL=claude-sonnet-4-6
CLAUDE_MAX_TOKENS=2048
```

`.gitignore` に `.env` を追記し、誤 push を防ぐ。

```bash
echo ".env" >> .gitignore
```

---

## anthropic ライブラリのセットアップ

```bash
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install anthropic python-dotenv
```

動作確認だけなら以下で十分だが、**本番コードにはこのまま使わない**（リトライもコスト記録もない）。

```python
import anthropic, os
from dotenv import load_dotenv

load_dotenv()
client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])
msg = client.messages.create(
    model=os.environ["CLAUDE_MODEL"],
    max_tokens=64,
    messages=[{"role": "user", "content": "こんにちは"}]
)
print(msg.content[0].text)
```

---

## リトライ・コスト追跡付きベース呼び出しクラス

本番パイプラインでは `ClaudeClient` クラスに一本化する。429/529 に自動リトライし、累計トークン費用を `.jsonl` に記録する。

```python
# src/claude_client.py
import os, time, json, logging
from datetime import datetime
from pathlib import Path
from dotenv import load_dotenv
import anthropic

load_dotenv()

COST_PER_M = {          # 2025-06 時点の入力/出力単価 (USD/1M tokens)
    "claude-sonnet-4-6": (3.0, 15.0),
    "claude-haiku-4-5-20251001": (0.8, 4.0),
}
LOG_PATH = Path("data/cost_log.jsonl")

class ClaudeClient:
    def __init__(self):
        self.client = anthropic.Anthropic(
            api_key=os.environ["ANTHROPIC_API_KEY"]
        )
        self.model = os.environ.get("CLAUDE_MODEL", "claude-sonnet-4-6")
        self.max_tokens = int(os.environ.get("CLAUDE_MAX_TOKENS", 2048))
        LOG_PATH.parent.mkdir(parents=True, exist_ok=True)

    def call(self, prompt: str, system: str = "", retries: int = 3) -> str:
        messages = [{"role": "user", "content": prompt}]
        kwargs = dict(model=self.model, max_tokens=self.max_tokens,
                      messages=messages)
        if system:
            kwargs["system"] = system

        for attempt in range(retries):
            try:
                resp = self.client.messages.create(**kwargs)
                self._log_cost(resp.usage)
                return resp.content[0].text
            except anthropic.RateLimitError:
                wait = 2 ** attempt * 10      # 10s → 20s → 40s
                logging.warning(f"RateLimit, wait {wait}s")
                time.sleep(wait)
            except anthropic.APIStatusError as e:
                if e.status_code == 529:      # API過負荷
                    time.sleep(30)
                else:
                    raise
        raise RuntimeError("Claude API failed after retries")

    def _log_cost(self, usage):
        inp, out = COST_PER_M.get(self.model, (3.0, 15.0))
        cost_usd = (usage.input_tokens * inp + usage.output_tokens * out) / 1_000_000
        record = {
            "ts": datetime.utcnow().isoformat(),
            "model": self.model,
            "in": usage.input_tokens,
            "out": usage.output_tokens,
            "usd": round(cost_usd, 6),
        }
        with LOG_PATH.open("a", encoding="utf-8") as f:
            f.write(json.dumps(record, ensure_ascii=False) + "\n")
```

呼び出し側は 2 行で済む。

```python
from src.claude_client import ClaudeClient

client = ClaudeClient()
text = client.call("競馬予想ブログ記事を800字で書いて", system="あなたは競馬ライターです")
print(text)
```

---

## よくある落とし穴

| 症状 | 原因 | 対処 |
|------|------|------|
| `AuthenticationError` | `.env` が読まれていない | `load_dotenv()` を呼び出し前に置く |
| 常に同じ内容が返る | `max_tokens` が小さすぎて途中で切れる | 2048 以上に設定し `stop_reason` を確認 |
| コストが青天井になる | 呼び出しループにガードなし | 月次 `usd` 集計を `cron` で監視し閾値超でアラート |
| `RateLimitError` が頻発 | 無料枠 Tier-1 の分あたりリクエスト制限 | 指数バックオフ（上記実装）で対応、並列数を絞る |

`data/cost_log.jsonl` を毎朝集計すれば、記事 1 本あたりの API 費用が把握でき、1 記事 7 分・月 5 万円の採算計算が具体的に立てられる。
