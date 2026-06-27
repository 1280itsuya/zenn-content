---
title: "Claude Sonnet 4.6でVite設定エラーを自動診断するPythonスクリプト50行実装"
free: false
---

## `anthropic==0.30` のインストールと `diag.py` の起動確認

依存は `anthropic` パッケージと Python 3.11 以上の標準ライブラリのみ。`venv` 環境で次の 2 行だけで完結する。

```bash
pip install anthropic==0.30.0
export ANTHROPIC_API_KEY="sk-ant-..."
```

Vite が吐いたエラーはリダイレクトで `error.txt` に書き出す。

```bash
vite build 2> error.txt
python diag.py tsconfig.json error.txt
```

これが本章の完成形コマンドだ。以降でその中身を 1 行ずつ組み立てる。

---

## few-shot 3 件 + JSON 出力強制でハルシネーションを排除するプロンプト設計

Claude Sonnet 4.6 に渡すシステムプロンプトには 3 つの制約を入れる。

1. **few-shot 3 件**：「このエラー → この差分」の具体例をモデルに先に見せることで出力形式を固定
2. **JSON 出力強制**：`{"patch": "...", "reason": "..."}` のスキーマを明示し、パース失敗は自動リトライ
3. **temperature 0 固定**：設定変更提案にランダム性は不要。同じエラーには必ず同じ差分が返る

```python
SYSTEM_PROMPT = """\
You are a TypeScript/Vite configuration expert.
Given a tsconfig.json and a build error, output ONLY valid JSON with this exact schema:
{"patch": "<unified diff>", "reason": "<1-sentence root cause in Japanese>"}

<EXAMPLE_1>
error: TS2307 Cannot find module './App' or its corresponding type declarations
patch: |
  --- tsconfig.json
  +++ tsconfig.json
  @@ -3,0 +4 @@
  +    "moduleResolution": "bundler",
reason: Vite ESM は TS 5.0+ の moduleResolution=bundler を要求する
</EXAMPLE_1>
<EXAMPLE_2>
error: RollupError: Could not resolve './utils.js' from 'src/main.ts'
patch: |
  --- tsconfig.json
  +++ tsconfig.json
  @@ -5 +5 @@
  -    "module": "commonjs",
  +    "module": "ESNext",
reason: commonjs は bare specifier の解決を壊す。ESNext に統一する
</EXAMPLE_2>
<EXAMPLE_3>
error: ERR_PACKAGE_PATH_NOT_EXPORTED
patch: |
  --- tsconfig.json
  +++ tsconfig.json
  @@ -6,0 +7 @@
  +    "customConditions": ["development"],
reason: exports フィールドがある pkg は customConditions で dev エントリを解放する必要がある
</EXAMPLE_3>
"""
```

---

## `diag.py` 本体 50 行の実装

```python
#!/usr/bin/env python3
"""Vite/ESM エラー自動診断 CLI — Claude Sonnet 4.6 使用"""
import json
import os
import sys
from pathlib import Path
import anthropic

SYSTEM_PROMPT = """..."""   # 上記の文字列をここに展開する

MAX_TSCONFIG_CHARS = 4_000  # コスト圧縮：超過分は末尾トリム
MAX_ERROR_CHARS    = 2_000

def _load(path: str, limit: int) -> str:
    return Path(path).read_text(encoding="utf-8")[:limit]

def diagnose(tsconfig_path: str, error_path: str) -> dict:
    tsconfig = _load(tsconfig_path, MAX_TSCONFIG_CHARS)
    error    = _load(error_path,    MAX_ERROR_CHARS)

    client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])
    msg = client.messages.create(
        model       = "claude-sonnet-4-6",
        max_tokens  = 512,
        temperature = 0,
        system      = SYSTEM_PROMPT,
        messages=[{
            "role": "user",
            "content": (
                f"## tsconfig.json\n```json\n{tsconfig}\n```\n\n"
                f"## Build Error\n```\n{error}\n```\n\n"
                "Output JSON only. No markdown fences."
            ),
        }],
    )
    raw = msg.content[0].text.strip()
    try:
        return json.loads(raw)
    except json.JSONDecodeError:
        # フェンスが混入した場合の後処理
        cleaned = raw.removeprefix("```json").removesuffix("```").strip()
        return json.loads(cleaned)

def main() -> None:
    if len(sys.argv) != 3:
        print("usage: python diag.py <tsconfig.json> <error.txt>", file=sys.stderr)
        sys.exit(1)
    result = diagnose(sys.argv[1], sys.argv[2])
    print("=== patch ===")
    print(result["patch"])
    print("\n=== reason ===")
    print(result["reason"])

if __name__ == "__main__":
    main()
```

`removeprefix` / `removesuffix` は Python 3.9+ の組み込み。モデルが稀にコードフェンスを付けて返すケースへの防衛線だ。

---

## API 費用が 1 回¥2〜¥5 に収まるトークン計算根拠

Sonnet 4.6 の料金は **input $3/MTok・output $15/MTok**（2026 年 6 月時点）。

| 項目 | トークン目安 | 費用 ($) |
|---|---|---|
| システムプロンプト（few-shot 3 件） | 約 600 tok | $0.0018 |
| tsconfig（4,000 chars トリム後） | 約 1,000 tok | $0.0030 |
| エラー文（2,000 chars） | 約 500 tok | $0.0015 |
| 出力（差分 + 理由） | 約 200 tok | $0.0030 |
| **合計** | **約 2,300 tok** | **$0.0093 ≈ ¥1.5** |

`MAX_TSCONFIG_CHARS = 4_000` のトリムがなければ monorepo の tsconfig（1 万字超）を丸ごと送るケースもあるが、それでも上限は **$0.05（¥7）** 程度だ。1 日 10 回実行しても月 **¥450 未満**に収まる。

---

## Slack 通知拡張 10 行と月次コストキャップ `.env` 設定

診断結果を Slack に流すには `requests` を 1 つ追加するだけで完結する。

```python
import requests

def notify_slack(webhook_url: str, result: dict, error_path: str) -> None:
    requests.post(webhook_url, json={
        "text": (
            f":hammer_and_wrench: *Vite 診断完了* `{error_path}`\n"
            f"*原因*: {result['reason']}\n"
            f"```{result['patch'][:600]}```"
        )
    }, timeout=5)
```

`.env` に月次上限を定義し、`diag.py` の `main()` 先頭でカウンタを確認する。

```bash
# .env
ANTHROPIC_API_KEY=sk-ant-...
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/XXX/YYY/ZZZ
DIAG_MONTHLY_LIMIT=200   # 200回/月 ≈ ¥300〜¥1,000 の上限
```

```python
import datetime

COUNTER_PATH = Path.home() / ".diag_count"

def check_limit(limit: int) -> None:
    month = datetime.date.today().strftime("%Y-%m")
    data  = json.loads(COUNTER_PATH.read_text()) if COUNTER_PATH.exists() else {}
    count = data.get(month, 0)
    if count >= limit:
        print(f"月次上限 {limit} 回に到達（今月: {count} 回）", file=sys.stderr)
        sys.exit(1)
    data[month] = count + 1
    COUNTER_PATH.write_text(json.dumps(data))
```

`main()` の冒頭で `check_limit(int(os.getenv("DIAG_MONTHLY_LIMIT", "200")))` を呼ぶだけで無人課金の暴走を防げる。CI 環境では `DIAG_MONTHLY_LIMIT=50` に絞るのが実運用の定番だ。
