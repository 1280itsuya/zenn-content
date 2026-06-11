---
title: "第3章 path alias二重管理を撲滅:vite.config.tsとtsconfig.pathsをvite-tsconfig-pathsで一本化"
free: false
---

## エラー全文:`Failed to resolve import "@/utils"` は2ファイル定義のズレ

`@/`が絡む解決失敗は、`vite.config.ts`の`resolve.alias`と`tsconfig.json`の`paths`を二重管理していることが原因の9割を占める。まず実際に出るエラーはこれだ。

```text
[vite]: Rollup failed to resolve import "@/utils/format" from "src/pages/Home.tsx".
This is most likely unintentional. Could not resolve "@/utils/format"
```

`tsc --noEmit`は通るのにViteだけ落ちる場合、`tsconfig.json`に`paths`があり`vite.config.ts`に`alias`が無い。逆にエディタだけ赤線なら定義が片方しかない。原因を確定させるコマンド:

```bash
grep -n "@/" tsconfig.json vite.config.ts vitest.config.ts 2>/dev/null
```

## git diff 1個:`resolve.alias`を撤去して`vite-tsconfig-paths`へ寄せる

二重管理を撲滅する修正は、プラグイン導入とalias全削除を1コミットで行う。

```bash
npm i -D vite-tsconfig-paths@5
```

```diff
 // vite.config.ts
-import { fileURLToPath } from 'node:url'
 import { defineConfig } from 'vite'
 import react from '@vitejs/plugin-react'
+import tsconfigPaths from 'vite-tsconfig-paths'

 export default defineConfig({
-  plugins: [react()],
-  resolve: {
-    alias: {
-      '@': fileURLToPath(new URL('./src', import.meta.url)),
-    },
-  },
+  plugins: [react(), tsconfigPaths()],
 })
```

これで`tsconfig.json`の`paths`が唯一の正になる。新規aliasの追加が2ファイル→1ファイルになり、`resolve.alias`と`paths`の値ズレ起因の解決失敗はゼロになる。

## 唯一の正:`compilerOptions.paths`に`baseUrl`を必ず添える

`vite-tsconfig-paths`は`baseUrl`を基準に`paths`を解決する。`baseUrl`が無いと`Missing baseUrl in compilerOptions`相当で無言スキップされる。

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"]
    }
  }
}
```

以降aliasを増やすときは`paths`にキーを1行足すだけ。`vite.config.ts`は触らない。

## Vitestの`test.alias`は継承されない:`vitest.config.ts`にも1行

`vitest run`で`Cannot find module '@/utils'`が出るのは、Vitestがプラグインを別設定で読むため。

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import tsconfigPaths from 'vite-tsconfig-paths'

export default defineConfig({
  plugins: [tsconfigPaths()],
  test: { environment: 'jsdom' },
})
```

`vite.config.ts`と`vitest.config.ts`を分けている構成では両方に`tsconfigPaths()`を入れる。`test.alias`を手書きする旧手法は`paths`との二重管理が復活するため使わない。

## monorepo:`Could not resolve` の真因は`root`基準の`baseUrl`ズレ

pnpm workspace等で`packages/web`配下のViteを実行すると、`baseUrl: "."`がリポジトリ直下を指してしまい解決に失敗する。プラグインに参照する`tsconfig`を明示する。

```ts
// packages/web/vite.config.ts
import tsconfigPaths from 'vite-tsconfig-paths'

export default defineConfig({
  plugins: [
    tsconfigPaths({
      root: __dirname,
      projects: ['./tsconfig.json'],
    }),
  ],
})
```

`projects`で各パッケージの`tsconfig.json`を指定すれば、`baseUrl`がそのパッケージ基準になり`@/`が正しく解決される。これでルートと子の`baseUrl`衝突による`Could not resolve import`が消える。
