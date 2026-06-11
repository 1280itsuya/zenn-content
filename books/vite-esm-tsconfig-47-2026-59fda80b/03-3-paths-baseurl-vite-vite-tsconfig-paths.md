---
title: "第3章: paths/baseUrlのエイリアスがVite実行時だけ壊れる問題をvite-tsconfig-pathsで解消"
free: false
---

## 結論: pathsは`tsc`を通すだけ、Viteは`vite-tsconfig-paths`かresolve.aliasのどちらかを必ず別途必要とする

`tsconfig.json`の`paths`はTypeScriptの型解決専用で、Viteのバンドラ（esbuild/Rollup）には一切伝わらない。だから`tsc --noEmit`は緑なのに`vite dev`で`Failed to resolve import "@/utils"`が出る。再現する最小設定はこれ。

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  }
}
```

## エラー文 `[plugin:vite:import-analysis] Failed to resolve import "@/components/Button"` を1分で消す

逆引きの起点はこの文字列。`@/`始まりのimportで出たら、原因は100%「Viteにエイリアスがないだけ」。`vite-tsconfig-paths`を入れると`tsconfig.json`の`paths`をViteが自動で読む。

```bash
npm i -D vite-tsconfig-paths@5
```

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import tsconfigPaths from 'vite-tsconfig-paths'

export default defineConfig({
  plugins: [tsconfigPaths()],
})
```

これで`tsconfig.json`が唯一の正（single source of truth）になり、別ファイルへの二重定義が消える。

## resolve.aliasを手書きすべき2条件: モノレポ参照とビルド時間を5%削りたいとき

`vite-tsconfig-paths`はプラグインなので解決のたびに`tsconfig`走査が走る。エイリアスが2〜3個で固定なら、手書きの方が依存ゼロ・起動が軽い。判断軸は「`tsconfig`を別パッケージから`extends`しているか」。`extends`が絡むモノレポはプラグイン、単一アプリは手書きが安定する。

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import { fileURLToPath, URL } from 'node:url'

export default defineConfig({
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url)),
    },
  },
})
```

`fileURLToPath`を使うのはESM(`"type": "module"`)で`__dirname`が`undefined`になるため。`path.resolve(__dirname, ...)`はここで`ReferenceError: __dirname is not defined`を投げる。

## index省略で出る `Failed to resolve import "@/hooks"` の正体はディレクトリ解決

`@/hooks/index.ts`を`@/hooks`と書くと、tscは通すがViteは拡張子・index補完を自動でやらない場合がある。`resolve.extensions`を明示するか、importを省略しないルールに倒す。

```ts
// vite.config.ts (resolve内に追記)
resolve: {
  extensions: ['.ts', '.tsx', '.js', '.jsx', '.json'],
}
```

`barrelsby`等のbarrel自動生成を使っているなら、`index.ts`の有無をビルド前に検査する一行を入れておくと事故が減る。

```bash
test -f src/hooks/index.ts || echo "barrel欠落: @/hooks が壊れます"
```

## Vitest側の二重定義を断つ: `vite.config.ts`を共有しエイリアスを1ファイルに集約

テストで`Cannot find module '@/utils'`が出るのは、Vitestが別の`vitest.config.ts`を読み、そこにエイリアスが無いから。`mergeConfig`で`vite.config.ts`を継承すれば、エイリアス定義は1ファイルで完結する。

```ts
// vitest.config.ts
import { defineConfig, mergeConfig } from 'vitest/config'
import viteConfig from './vite.config'

export default mergeConfig(
  viteConfig,
  defineConfig({
    test: { environment: 'jsdom', globals: true },
  }),
)
```

これで「`tsconfig` → `vite.config` → `vitest.config`」のエイリアス情報が一本道になり、追加時の修正箇所が`tsconfig.json`1か所で済む。

## コピペ可能な最終構成: 3ファイルで`@/`統一・index省略・Vitest二重定義を同時解決

迷ったら下の組み合わせをそのまま貼る。プラグイン版を採用し、`resolve.alias`は書かない（二重管理を避ける）。

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import tsconfigPaths from 'vite-tsconfig-paths'

export default defineConfig({
  plugins: [tsconfigPaths({ projects: ['./tsconfig.json'] })],
})
```

```json
// tsconfig.json
{ "compilerOptions": { "baseUrl": ".", "paths": { "@/*": ["src/*"] } } }
```

`vitest.config.ts`は前節の`mergeConfig`版を使う。この3ファイルが揃えば、`@/`系の`Failed to resolve import`はdev・build・testのどこでも再発しない。
