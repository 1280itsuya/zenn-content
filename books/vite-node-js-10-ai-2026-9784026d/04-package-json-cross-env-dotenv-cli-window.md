---
title: "package.jsonスクリプト設計とcross-env/dotenv-cli──Windows/Linux/CIで壊れない移植可能構成"
free: false
---

## `&&`はWindowsのcmdで死ぬ──cross-env 7.0.3で即解決

Claude SDKやPlaywrightを組み込んだAI自動化プロジェクトで最も多い「自分の環境では動く」系障害の発生源がこのスクリプト書き方。

```
'NODE_ENV' is not recognized as an internal or external command,
operable program or batch file.
```

```jsonc
// ❌ Windows cmdで即死
"scripts": {
  "dev": "NODE_ENV=development vite",
  "ai:run": "NODE_ENV=production ts-node src/claude-agent.ts"
}
```

修正は cross-env を前置するだけ。`npm install -D cross-env@7.0.3` 後:

```jsonc
// ✅ Windows/Linux/CI 全対応
"scripts": {
  "dev": "cross-env NODE_ENV=development vite",
  "ai:run": "cross-env NODE_ENV=production ts-node src/claude-agent.ts",
  "test": "cross-env NODE_ENV=test playwright test"
}
```

`&&` 連鎖もcmdでは挙動が不定。直列/並列を明示したいなら `npm-run-all@4.1.5` の `run-s`（直列）/ `run-p`（並列）が可読性と移植性を両立する。

```jsonc
"scripts": {
  "build:all": "run-s clean build:ts build:vite",
  "dev:watch": "run-p watch:ts watch:vite"
}
```

## dotenv-cli 7.4 の優先順位バグ──本番.envが開発を上書きした実例

dotenv-cli の `-e` フラグは**後から指定したファイルが優先**される。この仕様を読み飛ばすと次のログに出会う。

```
ANTHROPIC_API_KEY loaded from .env.production (overwriting .env.local)
AuthenticationError: invalid x-api-key
```

```bash
# ❌ .env.production が .env.local を上書き
dotenv -e .env.local -e .env.production -- ts-node src/claude-agent.ts

# ✅ 後ろほど強い＝環境固有を後置
dotenv -e .env -e .env.local -e ".env.${NODE_ENV}" -- vite
```

Claude SDK + dotenv-cli の推奨スクリプト構成:

```jsonc
"scripts": {
  "dev":     "dotenv -e .env -e .env.local -- cross-env NODE_ENV=development vite",
  "ai:dev":  "dotenv -e .env -e .env.local -- cross-env NODE_ENV=development ts-node src/claude-agent.ts",
  "ai:prod": "dotenv -e .env -- cross-env NODE_ENV=production ts-node src/claude-agent.ts"
}
```

`.env.local` はGitignore対象にして開発者固有の `ANTHROPIC_API_KEY` をここに分離する運用が最小リスク。

## postinstall が `npm ci` で2回走る問題

`postinstall` に Playwright のブラウザDLを書くとCIキャッシュHIT時でも毎回実行され、2〜3分ロスが生じる。Claude SDKのpostinstallと衝突して二重実行になるケースも確認されている。

```jsonc
// ❌ GitHub Actions 毎回ブラウザDL
"scripts": {
  "postinstall": "playwright install"
}

// ✅ 明示的に分離してCIから直接呼ぶ
"scripts": {
  "postinstall": "echo 'run npm run setup:browsers manually'",
  "setup:browsers": "playwright install --with-deps chromium"
}
```

```yaml
# .github/workflows/ci.yml
- name: Cache Playwright browsers
  uses: actions/cache@v4
  with:
    path: ~/.cache/ms-playwright
    key: pw-${{ runner.os }}-${{ hashFiles('package-lock.json') }}

- name: Install browsers (cache miss only)
  if: steps.cache-playwright.outputs.cache-hit != 'true'
  run: npm run setup:browsers
```

## npm scripts vs Makefile──AI自動化プロジェクトの使い分け判断

| 判断軸 | npm scripts | Makefile |
|--------|-------------|----------|
| チーム全員 Node.js 使用 | ✅ | — |
| Windows サポート必要 | ✅（cross-env前提） | ❌（要WSL） |
| Python スクリプトも混在 | — | ✅ |
| タスク依存グラフが複雑 | ❌（run-s/run-p限界） | ✅ |
| CI とローカルで同一コマンド | △ | ✅（make使えるなら） |

Claude SDK + Playwright + Vite が混在するAI自動化プロジェクトは **npm scripts + npm-run-all** が現実解。Makefile に手を出すのは、Pythonの学習スクリプトを同一リポジトリで管理し始めてから。

## GitHub Actions に Secrets を流し込む最小 .env パターン

`.env` をリポジトリに含めずCIを回す最小構成:

```yaml
- name: Write .env from Secrets
  run: |
    echo "ANTHROPIC_API_KEY=${{ secrets.ANTHROPIC_API_KEY }}" >> .env
    echo "NODE_ENV=production" >> .env

- run: npm run ai:prod
```

`echo "KEY=VALUE" >> .env` でSecretsを書き出す方式は移植性が最高で、`${{ secrets.* }}` はActionsログで自動マスクされる。

ローカルは `.env.local`、CIはSecretsからechoで生成──この2系統を守れば `.env` のコミット事故を構造的に防げる。

## 移植可能 package.json 雛形──cross-env + dotenv-cli 完成形

```jsonc
{
  "scripts": {
    "dev":          "dotenv -e .env -e .env.local -- cross-env NODE_ENV=development vite",
    "build":        "cross-env NODE_ENV=production vite build",
    "ai:dev":       "dotenv -e .env -e .env.local -- cross-env NODE_ENV=development ts-node src/index.ts",
    "ai:prod":      "dotenv -e .env -- cross-env NODE_ENV=production ts-node src/index.ts",
    "test":         "dotenv -e .env -e .env.test -- cross-env NODE_ENV=test playwright test",
    "setup:browsers": "playwright install --with-deps chromium",
    "build:all":    "run-s build",
    "lint":         "eslint src --ext .ts,.tsx"
  },
  "devDependencies": {
    "cross-env":    "^7.0.3",
    "dotenv-cli":   "^7.4.2",
    "npm-run-all":  "^4.1.5"
  }
}
```

`.env` 優先順位（弱→強）: `.env` → `.env.local` → `.env.{NODE_ENV}`。この設計で Windows ローカル・Mac ローカル・GitHub Actions の3環境が同一スクリプトで動く。
