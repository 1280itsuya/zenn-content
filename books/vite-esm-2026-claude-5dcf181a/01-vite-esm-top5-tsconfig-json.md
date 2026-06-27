---
title: "Vite+ESM最頻出エラーTOP5とtsconfig.json即修正ワンライナー【無料試し読み】"
free: true
---

Zenn有料Book第1章（無料試し読み）を執筆します。

---

Vite 5.x + React 18 + TypeScript構成でよく踏む5件を選んだ。再現環境は Node 20.11 / Vite 5.2 / TypeScript 5.4。各エラーに「エラー全文・原因1行・tsconfig差分」の3点セットだけを置く。この形式が本書の骨格で、第2章以降の18エラーとClaude自動診断スクリプトも同じ型で揃っている。

## ❶ TS2307: Cannot find module '@/components/Button'

```
error TS2307: Cannot find module '@/components/Button'
or its corresponding type declarations.
```

**原因** `tsconfig.json` の `paths` 未定義。

```diff
 {
   "compilerOptions": {
+    "baseUrl": ".",
+    "paths": { "@/*": ["src/*"] }
   }
 }
```

`vite.config.ts` の `resolve.alias` と二重管理になるため `vite-tsconfig-paths` プラグインで一元化する。

## ❷ ERR_REQUIRE_ESM — `"module": "CommonJS"` が原因

```
Error [ERR_REQUIRE_ESM]: require() of ES Module
.../node_modules/chalk/source/index.js not supported.
```

**原因** CommonJS出力が ESM パッケージを `require()` しようとしている。

```diff
 {
   "compilerOptions": {
-    "module": "CommonJS",
+    "module": "ESNext",
+    "moduleResolution": "Bundler"
   }
 }
```

`package.json` に `"type": "module"` が抜けている場合は同時追加。

## ❸ `import.meta.env` が Jest で undefined — tsconfig 分離で解決

```
ReferenceError: Cannot access 'import.meta' before initialization
  at Object.<anonymous> (src/config.ts:3:23)
```

**原因** Jest（CommonJS）が `import.meta` を解釈できない。

```jsonc
// tsconfig.test.json（Jestだけ向ける）
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "module": "CommonJS",
    "moduleResolution": "Node"
  }
}
```

```ts
// jest.config.ts
export default {
  transform: {
    "^.+\\.tsx?$": ["ts-jest", { tsconfig: "tsconfig.test.json" }]
  }
};
```

Vitest へ乗り換えるとこの問題ごと消える。

## ❹ verbatimModuleSyntax — `import type` 強制エラー（TypeScript 5.0+）

```
error TS1484: 'Foo' is a type and must be imported using
a 'import type' specifier because 'verbatimModuleSyntax' is set to true.
```

**原因** TypeScript 5.0以降の `verbatimModuleSyntax: true` が型インポートを強制する。

```diff
-import { Foo } from './types'
+import type { Foo } from './types'
```

既存コードが多く一括対応が難しい場合のみ `tsconfig.json` 側を切る。

```diff
 {
   "compilerOptions": {
-    "verbatimModuleSyntax": true
+    "verbatimModuleSyntax": false
   }
 }
```

## ❺ `process` / `__dirname` が undefined — `@types/node` 未追加

```
error TS2304: Cannot find name 'process'.
error TS2304: Cannot find name '__dirname'.
```

**原因** `@types/node` 未インストール、または `types` 配列から外れている。

```bash
npm install -D @types/node
```

```diff
 {
   "compilerOptions": {
+    "types": ["node", "vite/client"]
   }
 }
```

---

この章で示した3点セット形式（エラー全文・原因・差分）は本書全23エラーに適用している。

**第2章以降の収録内容：**

- 残り18エラー（`isolatedModules` 衝突 / `declaration emit` 失敗 / Dynamic import型推論失敗 など）
- **Claude Sonnet 4.6 自動診断スクリプト** — エラー文を1コマンドでAPIに投げてtsconfig差分を返す（API費用¥2〜5/回）
- **GitHub Actions CIテンプレート** — `tsc --noEmit` + `vite build` を毎pushで自動検証

診断スクリプト単体での時短効果の試算は第2章冒頭に置いた。購入後すぐ動かせる。
