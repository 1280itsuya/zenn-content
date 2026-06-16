---
title: "PR差分→Claudeレビュー：プロンプト設計で誤検知率を43%→5%未満に下げた実装と全文公開"
free: false
---

## 失敗ログ：生diff投入で実測43%の誤検知を記録した初期実装

最初の実装は単純だった。`gh pr diff` の出力をそのままユーザーメッセージに貼り付け、「このdiffをレビューして」と送っただけ。20PRで検証した結果、誤検知率43%（86件中37件が不要な指摘）を記録した。

典型的な誤検知パターン：
- テストファイルのfixture追加を「未使用変数のメモリリーク」と判定
- `poetry.lock` の差分を「依存関係の脆弱性」と警告
- 設定ファイルのコメント削除を「ドキュメント不足」と報告

```python
# NG：初期実装（誤検知43%）
import subprocess, anthropic

diff_text = subprocess.run(
    ["gh", "pr", "diff", str(pr_number)], capture_output=True, text=True
).stdout

client = anthropic.Anthropic()
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=2048,
    messages=[{"role": "user", "content": f"このdiffをレビューして:\n{diff_text}"}]
)
```

原因は3つ：ロール未定義でClaudeが「何でも拾う」動作をする、ロックファイルや生成コードを含むノイズdiff、自由形式テキストで「それっぽい指摘」が混入すること。

## 改善Step1：systemプロンプトのロール定義でSonnet 4.6の精度を43%→18%に引き上げる

「何を報告すべきでないか」を明示的に除外リストで定義することで、誤検知率が43%→18%に下がった。

```python
SYSTEM_PROMPT = """あなたはPythonバックエンドAPIの上級エンジニアです。
PRの差分のみに基づき、以下の条件を満たす問題だけを報告してください。

報告対象：
- バグ（ロジックエラー、型ミスマッチ、例外処理漏れ）
- セキュリティ（SQLインジェクション、認証バイパス、シークレットのハードコード）
- パフォーマンス（N+1クエリ、不要なループ内DB呼び出し）

報告しないもの：
- スタイル・命名規則・コメントの有無
- テストファイル内の変更
- 設定ファイル・ロックファイル・自動生成コード
- 確信度が70%未満の推測
"""
```

## 改善Step2：diff前処理でコンテキスト長を812行→271行に削減

ロックファイルと自動生成ファイルを除外し、追加/削除行のみを抽出する前処理を追加した。平均diff行数が812行→271行（1/3以下）に圧縮された。

```python
import re

SKIP_PATTERNS = [
    r"poetry\.lock$", r"package-lock\.json$", r"yarn\.lock$",
    r".*\.min\.(js|css)$", r".*_pb2\.py$",  # protobuf生成コード
]

def preprocess_diff(raw_diff: str, max_lines: int = 300) -> str:
    chunks = raw_diff.split("diff --git ")
    filtered_chunks = []
    for chunk in chunks[1:]:
        filename = re.search(r"a/(.+?) b/", chunk)
        if filename and any(re.search(p, filename.group(1)) for p in SKIP_PATTERNS):
            continue
        lines = chunk.split('\n')
        relevant = [l for l in lines if l.startswith(('+', '-', '@@', 'diff', '---', '+++'))]
        filtered_chunks.append('\n'.join(relevant))

    result = '\n'.join(filtered_chunks)
    return '\n'.join(result.split('\n')[:max_lines])
```

## JSON schema強制で構造化コメント出力、誤検知率を5%未満に収めた最終プロンプト全文

JSON出力を強制することで「根拠の薄い指摘」をツール検証層でフィルタリングできる。`confidence >= 0.7` の条件を組み合わせた結果、誤検知率は最終的に5%未満（20PR中1件）に収束した。

```python
REVIEW_TOOL = {
    "name": "report_issues",
    "description": "発見した問題を報告する。問題がなければ空のissues配列を返す。",
    "input_schema": {
        "type": "object",
        "properties": {
            "issues": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "file":       {"type": "string"},
                        "line":       {"type": "integer"},
                        "severity":   {"type": "string", "enum": ["critical", "warning"]},
                        "message":    {"type": "string", "maxLength": 200},
                        "confidence": {"type": "number", "minimum": 0, "maximum": 1}
                    },
                    "required": ["file", "line", "severity", "message", "confidence"]
                }
            }
        },
        "required": ["issues"]
    }
}

def review_diff(diff: str) -> list[dict]:
    client = anthropic.Anthropic()
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=4096,
        system=SYSTEM_PROMPT,
        tools=[REVIEW_TOOL],
        tool_choice={"type": "any"},
        messages=[{"role": "user", "content": f"以下のdiffをレビューしてください:\n\n{diff}"}]
    )
    for block in response.content:
        if block.type == "tool_use" and block.name == "report_issues":
            return [i for i in block.input["issues"] if i["confidence"] >= 0.7]
    return []
```

## ファイル種別routingでAPI呼び出し数を62%削減するrouting設計

全ファイルをフルレビューすると、テストファイルだけで平均API呼び出しの31%を占めていた。パターンマッチによるrouting設計を導入し、呼び出し数を62%削減した。

```python
from enum import Enum

class ReviewDepth(Enum):
    SKIP  = "skip"   # APIを呼ばない
    LIGHT = "light"  # セキュリティチェックのみ
    FULL  = "full"   # 全項目チェック

ROUTING_RULES: list[tuple[str, ReviewDepth]] = [
    (r"(test_|_test)\.(py|ts)$", ReviewDepth.SKIP),
    (r"\.(md|rst|txt)$",         ReviewDepth.SKIP),
    (r"\.(lock|sum)$",           ReviewDepth.SKIP),
    (r"\.(json|yaml|yml|toml)$", ReviewDepth.LIGHT),
    (r"\.(py|ts|go|rs)$",        ReviewDepth.FULL),
]

def get_review_depth(filename: str) -> ReviewDepth:
    for pattern, depth in ROUTING_RULES:
        if re.search(pattern, filename):
            return depth
    return ReviewDepth.LIGHT
```

## GitHub REST APIでPRへレビューコメントを自動投稿するPythonコード

```python
import os, requests

def post_pr_review(pr_number: int, issues: list[dict]) -> None:
    token = os.environ["GITHUB_TOKEN"]
    repo  = os.environ["GITHUB_REPOSITORY"]  # "owner/repo" 形式

    comments = [
        {
            "path": issue["file"],
            "line": issue["line"],
            "side": "RIGHT",
            "body": (
                f"**[{issue['severity'].upper()}]** {issue['message']}\n\n"
                f"_確信度: {issue['confidence']:.0%} | 自動検出: claude-sonnet-4-6_"
            )
        }
        for issue in issues
    ]

    payload = (
        {"event": "APPROVE", "body": "自動レビュー完了: 問題なし"}
        if not comments else
        {
            "event": "REQUEST_CHANGES",
            "body": f"{len(comments)}件の問題を検出しました（claude-sonnet-4-6）",
            "comments": comments
        }
    )

    resp = requests.post(
        f"https://api.github.com/repos/{repo}/pulls/{pr_number}/reviews",
        headers={
            "Authorization": f"Bearer {token}",
            "Accept": "application/vnd.github+json",
            "X-GitHub-Api-Version": "2022-11-28"
        },
        json=payload,
        timeout=30
    )
    resp.raise_for_status()
    print(f"レビュー投稿完了: {resp.json()['html_url']}")
```

GitHub Actions内では `GITHUB_TOKEN` が自動注入される。PRへの書き込みを許可するには、workflowファイルのpermissionsセクションに `pull-requests: write` を追加すること。問題ゼロの場合でも `APPROVE` イベントを投稿することで、レビュー済みステータスがGitHub上に明示され、マージ要件のステータスチェックとして機能する。
