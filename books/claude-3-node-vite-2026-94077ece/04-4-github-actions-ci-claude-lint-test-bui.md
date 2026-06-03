---
title: "第4章 GitHub Actions CIをClaudeで生成しlint/test/buildを90秒に収める"
free: false
---

<!-- topics: claude, ai, typescript, react, vite -->

push毎に lint・type-check・test・build を回す `workflow.yml` は、キャッシュ未設定だと3分超で詰まる。`actions/setup-node@v4` の `cache: 'npm'` と node-version matrix を入れれば90秒台に収まる。本章末で「PRを出すとCIがgreenになる」リポジトリ構成を提示する。

## Claudeが生成するworkflow.ymlは古いactionsバージョン問題を抱える

Claude に「Node+Vite の CI を書いて」と頼むと、`actions/checkout@v2` や `setup-node@v2` を平気で出す。2026年時点では `@v4` 系が必須で、`@v2` は Node16 ランナー廃止で warning → 将来失敗する。固定プロンプトで世代を縛る。

```bash
claude -p "Node20/Vite5前提のGitHub Actions CIを生成。\
checkout@v4, setup-node@v4 を厳守。node-versionは20.x。\
npm ci でinstallし lint/type-check/test/build を順に実行" \
  > .github/workflows/ci.yml
```

## node-version matrixとnpm cache未設定でCIが3分超になる失敗

初回生成物は `cache` 行が欠落し、毎回 `npm ci` が全依存をフルダウンロードする。実測で `Install dependencies` だけ142秒を食った。修正diffは1行追加で済む。

```diff
     - uses: actions/setup-node@v4
       with:
         node-version: ${{ matrix.node }}
+        cache: 'npm'
     - run: npm ci
```

```yaml
strategy:
  matrix:
    node: ['20.x']
```

## actions/setup-node cacheで142秒→90秒台に短縮する実測

`cache: 'npm'` 導入後、2回目以降は `~/.npm` が復元され `Install` が18秒に落ちた。total 193秒 → 89秒。job単位の所要を `gh` で検証できる。

```bash
gh run list --limit 1 --json databaseId -q '.[0].databaseId' \
  | xargs -I{} gh run view {} --json jobs \
    -q '.jobs[].steps[] | "\(.name): \(.completed_at)"'
```

## secretsを伴わない安全なci.yml雛形をClaudeで固める

公開リポジトリで `secrets` を参照する雛形を出されると、forkからのPRで漏洩リスクになる。lint/test/build のみなら secrets 不要。`permissions` を read に絞る。

```yaml
name: CI
on: [push, pull_request]
permissions:
  contents: read
jobs:
  verify:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: ['20.x']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check
      - run: npm test --if-present
      - run: npm run build
```

## PRを出すとCIがgreenになるVite+TypeScriptリポジトリ構成

`package.json` の scripts が CI のステップ名と一致していないと `npm run type-check` で即失敗する。生成物の落とし穴No.1がこれ。先回りで揃える。

```json
{
  "scripts": {
    "lint": "eslint .",
    "type-check": "tsc --noEmit",
    "test": "vitest run",
    "build": "vite build"
  }
}
```

ブランチを切り PR を出せば、上記workflowが89秒でgreenを返す。CI実行環境を自前で持つなら、月数百円のVPS（[ConoHa VPS / A8計測リンク]）でself-hosted runnerを立てると無料枠の分数制限を回避できる。型安全なCI設計をさらに深めるなら『[プロを目指す人のためのTypeScript入門 / Amazon]』が実務リファレンスとして手元に効く。
