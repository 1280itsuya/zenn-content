---
title: "vite.config.tsの必須10設定──Claude SDK・Playwrightを共存させるoptimizeDeps完全ガイド"
free: false
---

## エラー⑥: `Failed to resolve import` ──resolve.alias 3行で根絶する

`@/utils/api` 形式のパスエイリアスを使うプロジェクトで Vite が起動直後にこのエラーを出す。原因は TypeScript の `tsconfig.json` に書いた `paths` が Vite のモジュール解決に伝わっていないこと。`vite-tsconfig-paths` を使うか、`resolve.alias` に直書きする。

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import path from 'path'

export default defineConfig({
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '~': path.resolve(__dirname, './src'),
    },
  },
})
```

`tsconfig.json` 側の `paths` と完全に一致させないと型は通るがランタイムで死ぬ。diff で確認する。

```diff
// tsconfig.json
{
  "compilerOptions": {
-   "baseUrl": ".",
+   "baseUrl": ".",
+   "paths": {
+     "@/*": ["./src/*"],
+     "~/*": ["./src/*"]
+   }
  }
}
```

---

## エラー⑦: `Top-level await is not available` ──build.target を `es2022` に上げる

Claude SDK (`@anthropic-ai/sdk`) は内部で top-level await を多用する。Vite の `build.target` デフォルト値は `modules`（= ES2015相当）であり、そのままビルドすると次のエラーが出る。

```
[vite] Build failed:
  Top-level await is not available in the configured target environment
```

修正は1行。

```ts
// vite.config.ts
export default defineConfig({
  build: {
    target: 'es2022',          // ← ここだけ
    sourcemap: true,
  },
})
```

`esnext` でも動くが、Safari 15 以下で動作不可になる。Claude SDK を使うユーザー層はほぼ開発者ツール用途のため `es2022` が現実的な落としどころ。

---

## エラー⑧: `node:fs is not defined` ──SSRターゲット設定と `ssr.noExternal`

Playwright の `chromium.launch()` を呼ぶモジュールをブラウザビルドに混ぜると `node:fs`、`node:path`、`node:child_process` などが未定義になる。これはバンドラがブラウザターゲットとしてビルドしているため Node.js 組み込みモジュールをポリフィルしようとして失敗する。

```ts
// vite.config.ts
export default defineConfig({
  ssr: {
    noExternal: [
      '@playwright/test',
      'playwright-core',
      '@anthropic-ai/sdk',     // Node.js fetch実装を含むため
    ],
    target: 'node',            // SSRビルドはNode.js向けに明示
  },
  optimizeDeps: {
    exclude: [
      '@playwright/test',      // Playwrightはpre-bundleから除外
    ],
  },
})
```

`ssr.noExternal` に追加すると Vite は該当パッケージを ESM 変換せずそのまま Node.js に渡す。`@anthropic-ai/sdk` はブラウザ向けフェッチとNode.js向けフェッチを内部で切り替えるため、SSRターゲットを明示しないと `fetch is not defined` に化けることがある。

---

## `optimizeDeps.include` が `anthropic` に必要な理由

Vite の事前バンドル（pre-bundling）は CommonJS パッケージを ESM に変換してキャッシュする。`@anthropic-ai/sdk` は ESM 配布だが、内部依存の `node-fetch` や `form-data` が CJS であるため、起動時に変換が走らないとランタイムで混在エラーになる。

```ts
// vite.config.ts
export default defineConfig({
  optimizeDeps: {
    include: [
      '@anthropic-ai/sdk',
      'node-fetch',
    ],
    exclude: [
      '@playwright/test',
    ],
  },
})
```

`vite dev` 起動時に `.vite/deps/` 以下にキャッシュが生成される。キャッシュが古い場合は `rm -rf node_modules/.vite` で強制再生成する。

```bash
rm -rf node_modules/.vite && vite dev
```

---

## Claude SDK × Playwright 共存 vite.config.ts 完成形テンプレート

以下が本章で解説した設定を全部組み込んだ、実際に動作確認済みの完成形。コピーして `src/vite-env.d.ts` の `/// <reference types="vite/client" />` と合わせて使う。

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import path from 'path'
import react from '@vitejs/plugin-react'

export default defineConfig(({ mode }) => ({
  plugins: [react()],

  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '~': path.resolve(__dirname, './src'),
    },
  },

  build: {
    target: 'es2022',
    sourcemap: mode === 'development',
    rollupOptions: {
      // Node.js専用パッケージをクライアントバンドルから除外
      external: [
        /^node:/,
        '@playwright/test',
        'playwright-core',
      ],
    },
  },

  optimizeDeps: {
    include: [
      '@anthropic-ai/sdk',
      'node-fetch',
    ],
    exclude: [
      '@playwright/test',
    ],
  },

  ssr: {
    noExternal: [
      '@anthropic-ai/sdk',
      '@playwright/test',
      'playwright-core',
    ],
    target: 'node',
  },

  // node:* を使うサーバーサイドコードのために必要
  define: {
    'process.env.NODE_ENV': JSON.stringify(mode),
  },
}))
```

この設定で Claude SDK の Streaming API 呼び出し、Playwright による E2E テスト実行、フロントエンドの React コンポーネントが1つのプロジェクトに共存できる。`mode` を使って `sourcemap` を切り替えているため、本番ビルドでソースマップが漏洩しない。
