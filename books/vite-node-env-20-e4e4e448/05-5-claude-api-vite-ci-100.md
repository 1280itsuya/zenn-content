---
title: "第5章 Claude API キー×Viteビルドの本番漏洩事故ゼロ設計 — CI 自動検出スクリプト100行全文掲載"
free: false
---

Zenn有料Book第5章を執筆します。

## 第5章 Claude API キー×Viteビルドの本番漏洩事故ゼロ設計 — CI 自動検出スクリプト100行全文掲載

---

## Viteが `VITE_` 変数をJSバンドルに丸ごと埋め込む仕組み

Viteのビルドは `import.meta.env` 経由でアクセス可能な変数を、ビルド時に**文字列リテラルとしてJSバンドルに静的展開**する。これはWebpackの `DefinePlugin` と同じ動作だが、Viteは `VITE_` プレフィックスを持つ変数を**自動的に公開対象**とするため、開発者が意識しなくても漏洩が起きる。

```bash
# ビルド後のdist/assets/index-XXXX.jsを検索すると変数値が丸見えになる
$ grep -r "sk-ant-" dist/
dist/assets/index-a1b2c3d4.js:...const n="sk-ant-api03-XXXXX"...
```

`vite build --mode production` を実行した瞬間、`.env.production` の `VITE_CLAUDE_API_KEY` は難読化なしでバンドルに焼き付く。`source map` を無効にしても文字列リテラルは残る。

---

## 3つの実事故パターン — OSSリポジトリの匿名事例

**パターン1: Claude APIキーの直接埋め込み**

npm上のViteテンプレートスターターで、セットアップ手順に「`.env`に`VITE_CLAUDE_API_KEY=sk-ant-...`を追加してください」と記載されたまま本番運用したケース。ユーザーがそのまま`git push`し、Vercelのビルドログ経由でキーが漏洩した。

```diff
# 誤った設定（.envファイル）
-VITE_CLAUDE_API_KEY=sk-ant-api03-XXXXXXXXXXXXXXXXXXXX

# 正しい分離
+# フロントエンド（公開可）
+VITE_APP_TITLE=My App
+
+# サーバーサイド専用（VITE_プレフィックスなし）
+CLAUDE_API_KEY=sk-ant-api03-XXXXXXXXXXXXXXXXXXXX
```

**パターン2: Stripeシークレットキーの混入**

`VITE_STRIPE_SECRET_KEY` として設定し、フロントエンドからStripe APIを直叩きする実装。公開鍵(`pk_`)ではなく秘密鍵(`sk_`)を`VITE_`につけた状態でデプロイした実例がGitHub上に複数存在する（2026年6月時点、検索クエリ: `VITE_STRIPE_SECRET_KEY language:dotenv`）。

**パターン3: Dependabot PRによる間接的な変数露出**

ライブラリのアップデートPRに対してCIが走る際、`secrets.CLAUDE_API_KEY`を`VITE_CLAUDE_API_KEY`として渡す設定が残っていると、PRのビルドアーティファクトにキーが含まれる。fork元からのDependabot PRは`secrets`にアクセスできない設計だが、設定ミスで`pull_request_target`を使っているとforkから親のsecretが読める。

```bash
# 危険な設定パターン（CI YAML内）
env:
  VITE_CLAUDE_API_KEY: ${{ secrets.CLAUDE_API_KEY }}  # ← 絶対NG
  CLAUDE_API_KEY: ${{ secrets.CLAUDE_API_KEY }}       # ← サーバー側のみOK
```

---

## CI自動検出スクリプト100行全文 — `env_secret_scanner.py`

`VITE_` プレフィックスつき変数にシークレットパターンが混入していないかを検出する。Claude/OpenAI/Stripe/AWS/GitHub PATの10パターンに対応。

```python
#!/usr/bin/env python3
"""
env_secret_scanner.py — VITE_変数へのシークレット混入を検出するCIスクリプト
使い方: python env_secret_scanner.py [プロジェクトルート]
終了コード: 0=クリーン, 1=検出あり
"""
import re
import sys
from pathlib import Path
from typing import NamedTuple

SECRET_PATTERNS: list[tuple[str, str]] = [
    (r"sk-ant-[A-Za-z0-9\-]{20,}", "Anthropic/Claude API key"),
    (r"sk-[A-Za-z0-9]{20,}", "OpenAI API key"),
    (r"sk_live_[A-Za-z0-9]{24,}", "Stripe live secret key"),
    (r"sk_test_[A-Za-z0-9]{24,}", "Stripe test secret key"),
    (r"pk_live_[A-Za-z0-9]{24,}", "Stripe live publishable key (server-side?)"),
    (r"AIza[A-Za-z0-9\-_]{35}", "Google API key"),
    (r"AKIA[A-Z0-9]{16}", "AWS Access Key ID"),
    (r"ghp_[A-Za-z0-9]{36}", "GitHub PAT (classic)"),
    (r"github_pat_[A-Za-z0-9_]{82}", "GitHub PAT (fine-grained)"),
    (r"xoxb-[0-9]+-[A-Za-z0-9\-]+", "Slack Bot Token"),
]

class Finding(NamedTuple):
    file: str
    line_no: int
    var_name: str
    pattern_name: str
    masked_value: str

def mask(val: str) -> str:
    return val[:4] + "****" + val[-4:] if len(val) > 8 else "****"

def scan_env_file(filepath: Path) -> list[Finding]:
    findings: list[Finding] = []
    try:
        lines = filepath.read_text(encoding="utf-8").splitlines()
    except (UnicodeDecodeError, PermissionError):
        return findings
    for i, line in enumerate(lines, 1):
        stripped = line.strip()
        if not stripped or stripped.startswith("#"):
            continue
        if not stripped.startswith("VITE_"):
            continue
        if "=" not in stripped:
            continue
        var_name, _, raw_value = stripped.partition("=")
        value = raw_value.strip().strip('"').strip("'")
        for pattern, label in SECRET_PATTERNS:
            if re.search(pattern, value):
                findings.append(Finding(
                    file=str(filepath),
                    line_no=i,
                    var_name=var_name.strip(),
                    pattern_name=label,
                    masked_value=mask(value),
                ))
    return findings

def find_env_files(root: Path) -> list[Path]:
    candidates: set[Path] = set()
    for glob in (".env", ".env.*", "*.env"):
        candidates.update(root.glob(glob))
        candidates.update(root.glob(f"**/{glob}"))
    return [
        p for p in candidates
        if ".git" not in p.parts
        and "node_modules" not in p.parts
        and p.is_file()
    ]

def main() -> int:
    root = Path(sys.argv[1]) if len(sys.argv) > 1 else Path(".")
    env_files = find_env_files(root)
    if not env_files:
        print("No .env files found — skipping scan.")
        return 0
    all_findings: list[Finding] = []
    for f in sorted(env_files):
        all_findings.extend(scan_env_file(f))
    if not all_findings:
        print(f"✅ {len(env_files)} file(s) scanned. No secrets detected in VITE_ vars.")
        return 0
    print(f"❌ SECRET LEAK DETECTED — {len(all_findings)} violation(s) in VITE_ variables\n")
    for f in all_findings:
        print(f"  File     : {f.file}:{f.line_no}")
        print(f"  Variable : {f.var_name}")
        print(f"  Matched  : {f.pattern_name}")
        print(f"  Value    : {f.masked_value}")
        print()
    print("Remediation: Remove VITE_ prefix from these variables.")
    print("  Server-side vars must NOT start with VITE_.")
    print("  Access them via your backend API, not import.meta.env.\n")
    return 1

if __name__ == "__main__":
    sys.exit(main())
```

ローカルでの動作確認:

```bash
# テスト用の危険な.envを作って検出を確認
echo 'VITE_CLAUDE_API_KEY=sk-ant-api03-XXXXXXXXXXXXXXXXXX' > .env.test
python env_secret_scanner.py .
# → ❌ SECRET LEAK DETECTED が表示されれば成功
rm .env.test
```

---

## GitHub Actions完全版YAML — Dependabot PRでもスキャンを強制実行

`pull_request` イベントと `push` の両方でスキャンが走り、DependabotからのPRでも検出が機能する設定。`pull_request_target` は使わず `pull_request` に統一することでsecret漏洩のリスクを回避する。

```yaml
# .github/workflows/env-secret-scan.yml
name: env-secret-scan

on:
  push:
    branches: ["main", "release/**"]
    paths:
      - ".env*"
      - "**/.env*"
      - "src/**"
  pull_request:
    paths:
      - ".env*"
      - "**/.env*"
      - "src/**"

permissions:
  contents: read

jobs:
  scan:
    name: VITE_ Secret Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Run env_secret_scanner
        run: python scripts/env_secret_scanner.py ${{ github.workspace }}

      # Viteビルドを実際に走らせてバンドル内を二重チェック
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - run: npm ci

      - name: Vite build (production)
        run: npx vite build --mode production
        env:
          # CIでは本物のキーを渡さない
          VITE_APP_TITLE: "CI Build"

      - name: Scan built bundle for leaked secrets
        run: |
          python - <<'EOF'
          import re, sys
          from pathlib import Path

          PATTERNS = [
              r"sk-ant-[A-Za-z0-9\-]{20,}",
              r"sk-[A-Za-z0-9]{20,}",
              r"sk_live_[A-Za-z0-9]{24,}",
              r"AKIA[A-Z0-9]{16}",
              r"ghp_[A-Za-z0-9]{36}",
          ]
          dist = Path("dist")
          found = False
          for js in dist.glob("**/*.js"):
              text = js.read_text(encoding="utf-8", errors="ignore")
              for p in PATTERNS:
                  if re.search(p, text):
                      print(f"❌ Secret pattern `{p}` found in {js}")
                      found = True
          sys.exit(1 if found else 0)
          EOF
```

Dependabotが送ってくるバージョンアップPRは `secrets` コンテキストにアクセスできないため、上記YAMLではビルド時に本物のキーを渡さない構成にしている。これにより「CIが通っているのに本番でキーが焼き込まれる」という状況を防ぐ。

---

## 既存プロジェクトへの即時適用手順 — 3コマンドで完了

```bash
# 1. スクリプトをプロジェクトに配置
mkdir -p scripts
curl -sSL https://raw.githubusercontent.com/your-org/vite-env-guard/main/env_secret_scanner.py \
  -o scripts/env_secret_scanner.py

# 2. ローカルでの全ファイルスキャン（CIに入れる前の動作確認）
python scripts/env_secret_scanner.py .

# 3. package.jsonにpre-buildフックとして追加
npm pkg set scripts.prebuild="python scripts/env_secret_scanner.py ."
```

`prebuild` に追加することで `npm run build` を実行するたびに自動スキャンが走り、**CIに到達する前にローカルで検出**できる。GitHub Actionsの課金時間を無駄にしないためにも、このローカルゲートは有効にしておく。

`.env.example` を運用している場合は以下の除外設定を `env_secret_scanner.py` の `find_env_files` に追加する:

```python
# .env.exampleはダミー値なのでスキャン対象から除外
return [
    p for p in candidates
    if ".git" not in p.parts
    and "node_modules" not in p.parts
    and p.name != ".env.example"   # ← この行を追加
    and p.is_file()
]
```
