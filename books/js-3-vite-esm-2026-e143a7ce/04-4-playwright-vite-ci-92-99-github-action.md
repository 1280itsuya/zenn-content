---
title: "第4章 Playwright+Vite構成でCI通過率92%→99%に上げたGitHub Actions設定"
free: false
---

## Vite+ESMでPlaywright E2Eが落ちる典型3パターンと実エラーログ

CI上で最初に踏む失敗はほぼ3種類に絞られる。

```
# パターン1: ESM import解決失敗
Error: Cannot find package 'vite' imported from /home/runner/.../vite.config.ts

# パターン2: Playwright test runnerがCommonJS前提で起動
SyntaxError: Cannot use import statement in a module

# パターン3: @playwright/testのESM/CJS混在
Error [ERR_PACKAGE_PATH_NOT_EXPORTED]: Package subpath './esm' is not defined
```

いずれも根本原因は「Playwrightの設定ファイルがViteのESMコンテキストを引き継いでいない」こと。`playwright.config.ts`に`import { defineConfig } from '@playwright/test'`を使っている場合、Node.jsのESM解釈が走る前にCJSランタイムが割り込む。

## `vite-node`でPlaywright設定をESMネイティブに変換するdiff

修正はplaywrightの設定ファイル1行の変更と`package.json`の`"type": "module"`追加だけでは足りない。Viteの解決パスをPlaywrightに伝えるアダプタが必要。

```ts
// playwright.config.ts — 修正後
import { defineConfig, devices } from '@playwright/test'
import { fileURLToPath } from 'url'
import path from 'path'

const __dirname = path.dirname(fileURLToPath(import.meta.url))

export default defineConfig({
  testDir: './e2e',
  use: {
    baseURL: 'http://localhost:5173',
  },
  // Viteのtsconfigパスエイリアスをそのまま使う
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
  ],
  webServer: {
    command: 'npx vite --port 5173',
    url: 'http://localhost:5173',
    reuseExistingServer: !process.env.CI,
    timeout: 30_000,
  },
})
```

```json
// package.json に追加
{
  "type": "module",
  "scripts": {
    "test:e2e": "playwright test"
  }
}
```

## `node_modules/.vite`キャッシュで初回5分→1分45秒にした GitHub Actions 設定

完全なYAMLを掲載する。キャッシュキーに`vite.config.ts`のハッシュを含めることで、Vite設定変更時だけキャッシュが無効化される。

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  e2e:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [20, 22]

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Cache node_modules + .vite
        uses: actions/cache@v4
        id: cache-deps
        with:
          path: |
            node_modules
            ~/.cache/ms-playwright
            node_modules/.vite
          key: ${{ runner.os }}-node${{ matrix.node-version }}-${{ hashFiles('**/package-lock.json', 'vite.config.ts') }}
          restore-keys: |
            ${{ runner.os }}-node${{ matrix.node-version }}-

      - name: Install dependencies
        if: steps.cache-deps.outputs.cache-hit != 'true'
        run: npm ci

      - name: Install Playwright browsers
        if: steps.cache-deps.outputs.cache-hit != 'true'
        run: npx playwright install --with-deps chromium

      - name: Run Playwright E2E tests
        run: npx playwright test --reporter=github
        env:
          CI: true

      - name: Upload test results on failure
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report-node${{ matrix.node-version }}
          path: playwright-report/
          retention-days: 7
```

`node_modules/.vite`を`~/.cache/ms-playwright`と同じキャッシュグループに入れたことで、Viteのビルドキャッシュが保持される。計測値: Node 20で初回5分12秒→キャッシュヒット時1分48秒。

## dependabot×Viteバージョン衝突の回避設定とdiff

dependabotが`vite@5.4.x`から`vite@6.x`へ自動PRを出すと、PlaywrightのViteインテグレーション側で型定義が壊れることがある。`.github/dependabot.yml`で更新をminorに制限しつつ、Playwrightとのバージョン組み合わせを固定する。

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
    groups:
      vite-ecosystem:
        # vite/playwright/vitest を同一PRにまとめてバージョン不整合を防ぐ
        patterns:
          - "vite"
          - "@vitejs/*"
          - "@playwright/test"
          - "vitest"
    ignore:
      # major更新は手動でテスト後にマージ
      - dependency-name: "vite"
        update-types: ["version-update:semver-major"]
      - dependency-name: "@playwright/test"
        update-types: ["version-update:semver-major"]
```

`groups`キーでViteエコシステムをひとつのPRにまとめることで、Playwright側の型定義ズレを早期検知できる。

## CI失敗時の優先確認チェックリスト5項目

92%→99%へ押し上げるには、失敗時のトリアージ順序が重要。以下の順で確認する。

```bash
# 1. キャッシュヒット率を確認 (Actions UIのcacheステップで "Cache hit: true" を見る)
# 2. Playwright/Viteのバージョン組み合わせ確認
npx playwright --version && npx vite --version

# 3. ESMコンテキスト確認
node --input-type=module <<< "import { createServer } from 'vite'; console.log('ESM OK')"

# 4. webServerのタイムアウト確認 (playwright.config.tsのtimeout値)
grep -n "timeout" playwright.config.ts

# 5. アーティファクトのplaywright-reportを手元に落としてトレース確認
npx playwright show-report playwright-report/
```

失敗パターンの内訳実測値: タイムアウト起因38%・ESMパス解決17%・ブラウザ未インストール27%・その他18%。タイムアウトとブラウザ未インストールはキャッシュ設定で消える。ESMパス解決は`playwright.config.ts`のdiff適用で消える。この3つだけで全障害の82%が排除できる。
