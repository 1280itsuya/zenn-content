---
title: "第2章 Node.js 18/20/22バージョン競合でClaude SDKが壊れる3パターンとnvm固定手順"
free: false
---

```markdown
---
title: "Node.js 18/20/22バージョン競合でClaude SDKが壊れる3パターンとnvm固定手順"
topics: ["typescript", "claude", "nodejs", "automation", "javascript"]
---

## 3パターン症状早見表 — まず自分のエラーを特定する

エラーメッセージで該当節に直アクセスする。

| # | ターミナルに出るメッセージ | 根本原因 |
|---|---|---|
| ① | `ERR_MODULE_NOT_FOUND` (require時) | Claude SDK v0.28以降がESM-only化 |
| ② | `SyntaxError: Cannot use import statement` | `package.json` の `"type": "module"` 欠落 |
| ③ | `streamText is not iterable` / `async generator undefined` | グローバルNode.js ≠ プロジェクトローカルNode.js |

## パターン①: Node.js 18でrequireがERR_MODULE_NOT_FOUNDを返す

Claude SDK `@anthropic-ai/sdk` はv0.28からESM-onlyパッケージに移行した。Node.js 18の `require()` でCJSアダプタを探しに行くが存在しないため、Module Not Foundになる。

```bash
# 実ターミナルログ（Node 18.19.0 + @anthropic-ai/sdk 0.28.0）
$ node index.js
Error [ERR_MODULE_NOT_FOUND]: Cannot find module
  '/path/to/node_modules/@anthropic-ai/sdk/dist/index.cjs'
    at require (node:internal/modules/cjs/loader:1356:15)
```

修正は `nvm use 20` 一発 + ESM宣言の追加。

```bash
# Node.js 20に切り替え
nvm install 20 && nvm use 20

# package.json に "type": "module" を追記
node -e "
  const fs = require('fs');
  const p = JSON.parse(fs.readFileSync('./package.json','utf8'));
  p.type = 'module';
  fs.writeFileSync('./package.json', JSON.stringify(p, null, 2));
  console.log('Done:', p.type);
"

# 動作確認
node -e "import('@anthropic-ai/sdk').then(m => console.log('SDK OK:', Object.keys(m)))"
```

## パターン②: ESM/CJS混在でSyntaxError — package.jsonの"type"フィールド診断

`import Anthropic from '@anthropic-ai/sdk'` と書いているのに SyntaxError が出る場合、原因は `package.json` の `"type"` 未設定か、`.js` 拡張子でのimport混在のどちらかに絞られる。

```bash
# 実ターミナルログ
$ node tool.js
SyntaxError: Cannot use import statement outside a module
    at wrapSafe (node:internal/modules/cjs/loader:1488:18)

# 原因を1コマンドで判定
node -e "console.log(require('./package.json').type || '❌ type未定義=CJS扱い')"
```

修正は2択。既存の `require()` が多い場合は方針Bで `.mjs` を分離するほうが速い。

```jsonc
// 方針A: プロジェクト全体をESMに統一（新規プロジェクト推奨）
// package.json
{
  "type": "module"
}
```

```bash
# 方針B: Claudeを呼ぶファイルだけ .mjs に変更（最小侵襲）
mv tool.js tool.mjs
node tool.mjs   # SyntaxError が消える
```

## パターン③: グローバルNode.js 22 × ローカル18でasync generatorが壊れる

`claude.messages.stream()` のような async generator は Node.js 18.x 以下でイテレータプロトコルの実装が不完全で壊れる。nvm で `use 20` したつもりでも、新しいシェルを開くとグローバルに戻るのが落とし穴。

```bash
# 実ターミナルログ（グローバル: node 22 / nvm local: 18.12.0）
$ nvm use 18 && node stream.js
TypeError: (intermediate value)[Symbol.asyncIterator] is not a function
    at processTicksAndRejections (node:internal/process/task_queues:95:5)

# which node でnvmが効いていないことを確認
$ which node
/usr/local/bin/node   # ← nvmを経由していない
$ nvm current
v18.12.0              # ← nvm は 18 と言っているが PATH が別
```

現状確認コマンドで3行の出力が一致しなければ不一致確定。

```bash
node --version          # グローバルのバージョン
nvm current             # nvmが管理するバージョン
cat .nvmrc 2>/dev/null || echo "❌ .nvmrc 未作成"
```

## .nvmrc / .node-version / volta pin 3択比較と副業ツール向け推奨

| ツール | 固定ファイル | 自動切替 | CI対応 | 学習コスト |
|---|---|---|---|---|
| **nvm** | `.nvmrc` | shell hook で可 | ◎ | 低 |
| **nodenv** | `.node-version` | 自動 | ◎ | 低 |
| **volta** | `package.json` 内 | 完全自動 | ◎ | 中 |

ソロ副業ツール開発には `nvm + .nvmrc` が最小コストで最大互換。voltaは複数Node.jsを日常的に切り替えるチーム開発向け。

```bash
# nvm: .nvmrc を作成してプロジェクトに固定
echo "20.14.0" > .nvmrc
nvm use   # .nvmrc を自動読み取り

# volta: package.json の "volta" フィールドに自動追記
volta pin node@20.14.0

# nodenv: .node-version に書くだけ
echo "20.14.0" > .node-version
```

## GitHub Actions / CircleCI / Vercelのバージョン固定コマンド一覧

ローカルの `.nvmrc` をCIにそのまま読ませれば乖離ゼロになる。

```yaml
# GitHub Actions (.github/workflows/claude-tool.yml)
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version-file: ".nvmrc"   # .nvmrc から自動解決
      - run: npm ci && node index.js
```

```yaml
# CircleCI (.circleci/config.yml)
jobs:
  build:
    docker:
      - image: cimg/node:20.14
    steps:
      - checkout
      - run: node --version && npm ci && node index.js
```

```json
// Vercel (vercel.json)
{
  "build": {
    "env": {
      "NODE_VERSION": "20.14.0"
    }
  }
}
```

`.nvmrc` を1ファイル置くだけで、ローカル・GitHub Actions・Vercelのすべてが同じNode.js 20.14.0で動く。第1章の診断スクリプトが「`.nvmrc` 未作成」を検出した場合は、本章の `echo "20.14.0" > .nvmrc` コマンドを実行して再チェックすれば完了する。
```
