---
title: "jsconfig.json / tsconfig.json 地雷5種：Playwright と Claude SDK が壊れる設定diffと修復パッチ"
free: false
---

env-diag.jsが障害番号 **#4〜#8** を検出した場合、原因は `jsconfig.json` または `tsconfig.json` の設定にある。5パターンを修復diff付きで解剖する。

## 障害 #4: `moduleResolution: node` × Anthropic SDK 0.20+ で ERR_PACKAGE_PATH_NOT_EXPORTED

`@anthropic-ai/sdk` 0.20以降はESMサブパスエクスポートを使う。`moduleResolution: node` は `package.json#exports` を読まないため、内部パスへのアクセスが失敗する。

```diff
// tsconfig.json
{
  "compilerOptions": {
-   "moduleResolution": "node",
+   "moduleResolution": "bundler",   // Vite/esbuild 前提。ts-node なら "node16"
+   "module": "ESNext"
  }
}
```

```bash
# 修正確認
npx tsc --noEmit && echo "OK"
node -e "import('@anthropic-ai/sdk').then(() => console.log('ESM import OK'))"
```

`bundler` は Vite/esbuild 前提。ts-nodeで単体実行するなら `node16` を選ぶ。

## 障害 #5: `paths` エイリアスが ts-node に無視されて Cannot find module '@/lib/...'

TypeScriptの `paths` はコンパイル時の型解決のみ担う。ts-nodeはランタイムに `paths` を読まないため、`tsconfig-paths` の明示的な登録が必須。

```bash
npm install -D tsconfig-paths
```

```diff
// package.json scripts
- "dev": "ts-node src/index.ts"
+ "dev": "ts-node -r tsconfig-paths/register src/index.ts"
```

```typescript
// playwright.config.ts — Playwright 実行時も先頭行に追加
import 'tsconfig-paths/register';
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './src/tests',
});
```

`tsconfig-paths/register` を `playwright.config.ts` の先頭に置かないと、テスト実行時のエイリアス解決が ts-node の問題と同じ経路でコケる。

## 障害 #6: `strict: true` × `@playwright/test` 旧型定義の TS2345 大量発生

`strict: true` は `strictNullChecks` / `noImplicitAny` を一括有効化する。古い `@playwright/test` の型定義に残った `any` と衝突し `TS2345` が数十件出てビルドが止まる。

```bash
# まず @playwright/test を最新に上げる
npm install -D @playwright/test@latest
npx tsc --noEmit
```

それでも残るなら `strict` を一括適用せず個別フラグで段階移行する:

```diff
// tsconfig.json
{
  "compilerOptions": {
-   "strict": true,
+   "strictNullChecks": true,
+   "noImplicitAny": true
  }
}
```

`strict: true` は依存の型定義が最新である前提。レガシー依存が混在するプロジェクトでは個別制御が安全。

## 障害 #7: `target: ES5` で Playwright の async/await がタイムアウト多発

`target: ES5` はasync/awaitをPromiseチェーンに変換する。Playwrightの `page.waitForSelector` 等は内部でMicrotaskキューのタイミングを前提とするため、ES5変換後のPolyfillと干渉してタイムアウトが頻発する。

```diff
// tsconfig.json
{
  "compilerOptions": {
-   "target": "ES5",
+   "target": "ES2020",
    "lib": ["ES2020", "DOM"]
  }
}
```

```typescript
// 修正後の確認用テスト（5秒以内に通過すれば障害 #7 は解消）
import { test, expect } from '@playwright/test';

test('async timing regression check', async ({ page }) => {
  await page.goto('https://example.com');
  const h1 = await page.locator('h1').first().textContent();
  expect(h1).toBeTruthy();
});
```

```bash
npx playwright test --timeout=5000
```

Node.js 14以降 / Chromiumベース環境はES2020ネイティブで動く。ES5に下げる理由がなければ `ES2020` 固定でよい。

## 障害 #8: `rootDir`/`outDir` 未設定で claude CLI の bin パスが解決失敗

`claude` CLIはグローバルインストール時に `package.json#bin` で `dist/cli.js` を参照する。`outDir` 未設定だと `tsc` がソースと同ディレクトリに `.js` を吐き、`bin` パスが存在せず `command not found` になる。

```diff
// tsconfig.json
{
  "compilerOptions": {
+   "rootDir": "src",
+   "outDir": "dist",
    "declaration": true
  },
+ "include": ["src/**/*"],
+ "exclude": ["node_modules", "dist"]
}
```

```bash
# ビルド後に bin パスが生成されているか確認
npx tsc
ls dist/cli.js

# package.json の bin エントリと突合
node -e "const p = require('./package.json'); console.log(JSON.stringify(p.bin, null, 2))"
```

```bash
# 5パターン修復後に env-diag.js を再実行して全 PASS を確認
node env-diag.js | grep -E "^#[4-8]"
# 期待出力:
# #4 PASS moduleResolution
# #5 PASS tsconfig-paths
# #6 PASS strict flags
# #7 PASS target ES2020
# #8 PASS outDir dist
```

5パターンすべて `tsconfig.json` の1〜3行変更で解決する。`env-diag.js` を再実行して `#4 PASS` 〜 `#8 PASS` が揃ったら次章へ進む。
