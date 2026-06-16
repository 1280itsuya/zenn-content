---
title: "pytest失敗ログをClaude tool_useで解析し修正Branchを30秒で自動生成する実装"
free: false
---

## pytest --tb=long の出力をJSONに整形してClaude APIへ投げる

`pytest`がデフォルトの`--tb=short`だとスタックトレースが途中で切れ、Claude側でどのローカル変数が問題かを判断できない。`--tb=long`を強制指定することで正答率が5回中3回→4回に上がった（後述）。

以下のスクリプトは pytest の終了コードが非0のとき stdout を構造化して返す。

```python
# scripts/run_pytest.py
import subprocess
import json
import sys

def run_and_capture(test_path: str = "tests/") -> dict:
    result = subprocess.run(
        ["pytest", test_path, "--tb=long", "--no-header", "-q"],
        capture_output=True,
        text=True,
    )
    return {
        "exit_code": result.returncode,
        "stdout": result.stdout[-8000:],   # Claude context 節約: 末尾8000文字
        "stderr": result.stderr[-2000:],
    }

if __name__ == "__main__":
    data = run_and_capture(sys.argv[1] if len(sys.argv) > 1 else "tests/")
    print(json.dumps(data, ensure_ascii=False))
```

## tool_use Allowlistで「変更可能ファイル」を宣言して意図しない書き換えを封じる

Claude に自由にファイルを書かせると `requirements.txt` や CI 設定まで変更するケースがある。`tools` 定義の中に `allowed_paths` パラメータを必須フィールドとして要求し、Python 側で検証することでリスクを封じる。

```python
# scripts/claude_fixer.py
import anthropic
import json

TOOLS = [
    {
        "name": "apply_patch",
        "description": "指定ファイルにunified diff形式のパッチを適用する",
        "input_schema": {
            "type": "object",
            "properties": {
                "allowed_paths": {
                    "type": "array",
                    "items": {"type": "string"},
                    "description": "変更を許可するファイルパスのリスト (src/ or tests/ 配下のみ)",
                },
                "patch": {
                    "type": "string",
                    "description": "unified diff形式の修正パッチ",
                },
                "explanation": {"type": "string"},
            },
            "required": ["allowed_paths", "patch", "explanation"],
        },
    }
]

ALLOWED_PREFIX = ("src/", "tests/")

def validate_paths(paths: list[str]) -> None:
    for p in paths:
        if not any(p.startswith(prefix) for prefix in ALLOWED_PREFIX):
            raise ValueError(f"Blocked path: {p}  (allowed: {ALLOWED_PREFIX})")

def request_fix(pytest_output: dict) -> dict | None:
    client = anthropic.Anthropic()
    prompt = f"""以下のpytest失敗ログを解析し、apply_patch toolでコードを修正してください。
変更は src/ または tests/ 配下のファイルのみに限定してください。

{json.dumps(pytest_output, ensure_ascii=False)}"""

    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=4096,
        tools=TOOLS,
        messages=[{"role": "user", "content": prompt}],
    )
    for block in response.content:
        if block.type == "tool_use" and block.name == "apply_patch":
            validate_paths(block.input["allowed_paths"])
            return block.input
    return None
```

## Claude tool_use レスポンスを git apply → commit → push する実装

`validate_paths` を通過したパッチだけを `git apply` に渡す。失敗時はブランチを削除してクリーンアップする。

```python
# scripts/auto_branch.py
import subprocess
import tempfile
import os
from datetime import datetime

def create_fix_branch(patch: str, explanation: str) -> str:
    branch = f"auto-fix/{datetime.now().strftime('%Y%m%d-%H%M%S')}"
    subprocess.run(["git", "checkout", "-b", branch], check=True)
    with tempfile.NamedTemporaryFile(suffix=".patch", mode="w", delete=False) as f:
        f.write(patch)
        patch_file = f.name
    try:
        subprocess.run(["git", "apply", "--check", patch_file], check=True)
        subprocess.run(["git", "apply", patch_file], check=True)
        subprocess.run(["git", "add", "-p", "--", "src/", "tests/"], check=False)
        subprocess.run(["git", "add", "src/", "tests/"], check=True)
        subprocess.run(
            ["git", "commit", "-m", f"auto-fix: {explanation[:72]}"],
            check=True,
        )
        subprocess.run(["git", "push", "origin", branch], check=True)
    except subprocess.CalledProcessError:
        subprocess.run(["git", "checkout", "-"])
        subprocess.run(["git", "branch", "-D", branch])
        raise
    finally:
        os.unlink(patch_file)
    return branch
```

## GitHub Actions YAML全文：pytest失敗→修正Branch→PR作成を1ジョブで完走

```yaml
# .github/workflows/auto-fix.yml
name: auto-fix-on-pytest-failure

on:
  push:
    branches: ["main", "develop"]

jobs:
  test-and-fix:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.GITHUB_TOKEN }}

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - run: pip install -r requirements.txt pytest anthropic

      - name: Run pytest and capture output
        id: pytest
        run: |
          python scripts/run_pytest.py > pytest_result.json || true
          echo "exit_code=$(python -c "import json; print(json.load(open('pytest_result.json'))['exit_code'])")" >> $GITHUB_OUTPUT

      - name: Claude auto-fix
        if: steps.pytest.outputs.exit_code != '0'
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GIT_AUTHOR_NAME: "auto-fix-bot"
          GIT_AUTHOR_EMAIL: "bot@example.com"
          GIT_COMMITTER_NAME: "auto-fix-bot"
          GIT_COMMITTER_EMAIL: "bot@example.com"
        run: |
          python - <<'EOF'
          import json, subprocess
          from scripts.claude_fixer import request_fix
          from scripts.auto_branch import create_fix_branch

          pytest_output = json.load(open("pytest_result.json"))
          result = request_fix(pytest_output)
          if result is None:
              print("Claude: tool_use 未応答。手動対応が必要。")
              exit(0)

          branch = create_fix_branch(result["patch"], result["explanation"])
          subprocess.run([
              "gh", "pr", "create",
              "--title", f"[auto-fix] {result['explanation'][:60]}",
              "--body", f"Claude tool_use による自動修正\n\n```\n{result['explanation']}\n```",
              "--base", "main",
              "--head", branch,
          ], check=True)
          EOF

      - name: Upload pytest log as artifact
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: pytest-result
          path: pytest_result.json
```

`gh` CLI は `ubuntu-latest` に標準搭載済みのため追加インストール不要。`GITHUB_TOKEN` の `pull-requests: write` 権限で PR 作成まで完結する。

## 実測5回試行の定量レポートと誤修正1回の根本原因

| 試行 | 失敗テスト | 修正結果 | `--tb` オプション |
|:---:|---|---|:---:|
| 1 | `test_tax_calc.py::test_deduction` | 正しい修正 | long |
| 2 | `test_api.py::test_timeout` | 正しい修正 | long |
| 3 | `test_db.py::test_rollback` | 正しい修正 | long |
| 4 | `test_parser.py::test_encoding` | **誤修正** | short（実験） |
| 5 | `test_auth.py::test_expire` | 正しい修正 | long |

4回目だけ `--tb=short` にした実験で誤修正が発生した。`short` では `locals()` の変数ダンプが省略されるため、Claude が誤った変数スコープを推測した。`--tb=long` を強制する以外に、`pytest --tb=long -s 2>&1 | tail -200` でコンテキスト長を制限しつつ末尾優先で渡すのが現時点の最適解。

誤修正が実際に `git apply` されても、Allowlist (`src/`, `tests/`) 外のファイルは変更されない。さらに `auto-fix/` ブランチが main に直接マージされることはなく、PR レビューが人間ゲートとして機能する設計のため、誤修正の blast radius は pull request 1 件止まりに収まる。
