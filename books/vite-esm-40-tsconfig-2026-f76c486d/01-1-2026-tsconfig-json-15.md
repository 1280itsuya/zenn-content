---
title: "第1章 まずこれ貼れ:2026年版・全部入りtsconfig.json完成形と頻出エラー15件逆引き"
free: true
---

## 結論:この3ファイルを貼れば Vite+react-ts は8割直る

`npm create vite@latest -- --template react-ts` 直後に出る赤波線の大半は、`tsconfig` の3分割が壊れていることが原因。まず以下をルートの `tsconfig.json` に丸ごと上書きする。

```json
{
  "files": [],
  "references": [
    { "path": "./tsconfig.app.json" },
    { "path": "./tsconfig.node.json" }
  ],
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  }
}
```

ルートは振り分け専用。実体は `app` と `node` に書く。

## tsconfig.app.json と tsconfig.node.json の境界線(Vite 7 既定)

`src` 配下(ブラウザ実行)は `app`、`vite.config.ts` 等(Node 実行)は `node` が見る。`paths` は両方に書かないと alias が片方で死ぬ。

```jsonc
// tsconfig.app.json
{
  "compilerOptions": {
    "target": "ES2022",
    "moduleResolution": "bundler",
    "module": "ESNext",
    "jsx": "react-jsx",
    "strict": true,
    "noEmit": true,
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["src"]
}
```

```jsonc
// tsconfig.node.json
{
  "compilerOptions": {
    "moduleResolution": "bundler",
    "module": "ESNext",
    "composite": true,
    "noEmit": true
  },
  "include": ["vite.config.ts"]
}
```

## TS2307/TS5097/TS2792:import 解決系3件の逆引き diff

**`error TS2307: Cannot find module '@/lib/foo' or its corresponding type declarations.`**
原因:`paths` を書いたが Vite 側 alias が未設定。

```diff
// vite.config.ts
+import path from 'node:path'
 export default defineConfig({
   plugins: [react()],
+  resolve: { alias: { '@': path.resolve(__dirname, './src') } },
 })
```

**`error TS5097: An import path can only end with a '.ts' extension when 'allowImportingTsExtensions' is enabled.`**
原因:`import './a.ts'` と拡張子付きで書いた。

```diff
-import { sum } from './math.ts'
+import { sum } from './math'
```

**`error TS2792: Cannot find module 'react'. Did you mean to set the 'moduleResolution' option to 'nodenext'?`**
原因:`moduleResolution` が古い `node` のまま。

```diff
// tsconfig.app.json
-    "moduleResolution": "node",
+    "moduleResolution": "bundler",
```

## import.meta.env に型がない(TS2339)を vite-env.d.ts 1行で消す

**`error TS2339: Property 'env' does not exist on type 'ImportMeta'.`**
原因:`/// <reference>` 行が消えている、または独自 env 変数の型未定義。

```diff
// src/vite-env.d.ts
 /// <reference types="vite/client" />
+interface ImportMetaEnv {
+  readonly VITE_API_URL: string
+}
+interface ImportMeta { readonly env: ImportMetaEnv }
```

これで `import.meta.env.VITE_API_URL` が `string` 型で補完される。

## 残り25件と運用編で何が片付くか

ここまでの15件で、新規 `react-ts` プロジェクトの初日に踏む事故はほぼ収束する。一方、開発が進むと出る順に難所が変わる:

```text
TS6133 宣言したが未使用 → noUnusedLocals の運用
TS1259 esModuleInterop と default import の衝突
TS2769 vitest の型が引数に乗らない
TS2688 @types/node の取り違え(Cannot find type definition file)
verbatimModuleSyntax 有効化後の import type 一括置換
```

第2章以降は、この「出る順」で残り25件を同じ `エラー全文→原因1行→diff` 形式で並べ、最終章で monorepo・`project references` ビルドが詰まる運用編まで踏み込む。検索で飛んできた1件を3分で消すための逆引き辞書として、手元に置いて使い倒してほしい。
