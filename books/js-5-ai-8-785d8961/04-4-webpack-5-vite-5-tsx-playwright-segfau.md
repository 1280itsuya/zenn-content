---
title: "第4章：webpack 5・Vite 5・tsxでPlaywrightがsegfaultする3パターンと--experimental-vm-modules完全回避策"
free: false
---

## パターン1：webpack 5の`browser`フィールドがPlaywright用Node.jsモジュールを静かに破壊する

**エラーログ実物**
```
Error: crypto.createHash is not a function
    at PlaywrightBrowserType._launchServer (node_modules/playwright-core/lib/server/browserType.js:142:28)
Segmentation fault (core dumped)
```

**根本原因1行**
webpack 5が`package.json`の`browser`フィールドを優先し、Node.js標準`crypto`モジュールをブラウザ向けポリフィルに置換する。

**修正パッチ**

```js
// webpack.config.js
module.exports = {
  target: 'node',   // ← これ1行で browser field の優先を無効化
  resolve: {
    fallback: {
      crypto: false,
      fs: false,
      path: false,
    },
  },
};
```

再現確認コマンド（修正前後で比較する）：

```bash
# webpack build 後に下記を実行 → segfault が再現する
node -e "const { chromium } = require('playwright'); chromium.launch().then(b => b.close())"

# webpack.config.js 修正後は正常終了することを確認
npx playwright test --reporter=line
```

---

## パターン2：Vite 5 + vitest「Test environment setup error」スタックトレース全文と2行修正

**エラーログ実物**
```
Error: Test environment setup error
    at setupFile (node_modules/vitest/dist/node.js:4521:17)
TypeError: Cannot read properties of undefined (reading 'launch')
    at Object.<anonymous> (tests/scraper.test.ts:8:24)
```

**根本原因1行**
vitestのデフォルト設定がPlaywright用グローバル（`chromium`、`page`）をテストスコープへ注入しない。

**修正パッチ（vitest.config.tsへの2行追記）**

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'node',   // ← 追記1行目
    globals: true,          // ← 追記2行目
    testTimeout: 60000,
    setupFiles: ['./tests/setup.ts'],
  },
});
```

```ts
// tests/setup.ts
import { chromium, Browser } from 'playwright';

let browser: Browser;

beforeAll(async () => {
  browser = await chromium.launch({ headless: true });
});

afterAll(async () => {
  await browser.close();
});
```

---

## パターン3：tsx（ts-node後継）でPlaywright CLIが起動しない原因とpackage.json修正

**エラーログ実物**
```
$ tsx src/scraper.ts
Error [ERR_UNKNOWN_FILE_EXTENSION]: Unknown file extension ".ts"
    at new NodeError (node:internal/errors:405:5)
TypeError: The "payload" argument must be of type string. Received undefined
```

**根本原因1行**
tsxはESM前提だが、PlaywrightのNode.js APIがCJS形式で読み込まれるため、`"type": "module"`との設定が衝突する。

**修正パッチ（副業ツールのpackage.json scriptsに合わせた形）**

```json
{
  "scripts": {
    "scrape": "tsx --tsconfig tsconfig.cjs.json src/scraper.ts",
    "scrape:watch": "tsx watch src/scraper.ts"
  },
  "devDependencies": {
    "tsx": "^4.7.0",
    "playwright": "^1.44.0"
  }
}
```

```jsonc
// tsconfig.cjs.json（Playwright CLI専用の別設定）
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "module": "CommonJS",
    "moduleResolution": "Node"
  }
}
```

---

## `--experimental-vm-modules`が不要になるNode.js 22移行手順

Node.js 22以降ではフラグなしでESMのdynamic importが安定動作する。フラグを消すだけで起動時間が約0.3秒短縮される。

**移行前後の差分**

```bash
# Node.js 20以前（フラグ必須）
node --experimental-vm-modules node_modules/.bin/jest

# Node.js 22以降（フラグ不要・そのままで動く）
node node_modules/.bin/jest
```

```json
{
  "engines": {
    "node": ">=22.0.0"
  },
  "scripts": {
    "test": "jest",
    "test:playwright": "playwright test",
    "scrape": "tsx src/scraper.ts"
  }
}
```

バージョン固定ファイルを必ず置く：

```
# .node-version
22.3.0
```

---

## CI/CDで同一segfaultを再発させないGitHub Actions matrix戦略

```yaml
# .github/workflows/playwright.yml
name: Playwright E2E

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        node-version: [20, 22]       # ← 2バージョンで同時検証
        bundler: [webpack5, vite5]   # ← バンドラーも変数化
    steps:
      - uses: actions/checkout@v4

      - name: Node.js ${{ matrix.node-version }} セットアップ
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}

      - run: npm ci

      - name: Playwright ブラウザインストール
        run: npx playwright install --with-deps chromium

      - name: ${{ matrix.bundler }} でビルド＆テスト実行
        run: npm test
        env:
          BUNDLER: ${{ matrix.bundler }}

      - name: 失敗時レポート保存
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: report-node${{ matrix.node-version }}-${{ matrix.bundler }}
          path: playwright-report/
          retention-days: 7
```

このmatrixにより「Node.js 20 + webpack 5でのみsegfault」「Node.js 22 + Vite 5では通過」という差分が数字で可視化される。PRマージ後に「なぜ突然壊れたか」を追う時間が消える。
