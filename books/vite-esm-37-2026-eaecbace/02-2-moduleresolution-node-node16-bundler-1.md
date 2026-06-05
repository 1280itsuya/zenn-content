---
title: "第2章 moduleResolution: node/node16/bundler を取り違えた時の12エラー逆引き"
free: false
---

## "bundler" 設定下で出る ERR_UNKNOWN: relative import paths need explicit file extensions

Vite + `tsc` で最初に遭遇するのがこのエラー。原因は `moduleResolution: "node16"` または `"nodenext"` を指定し、相対 import の拡張子を省略していること。Vite のビルドには不要だが Node 実行ファイル（スクリプト・CLI）では必須になる。

```jsonc
// tsconfig.node.json — Node実行コード用
{
  "compilerOptions": {
    "module": "node16",
    "moduleResolution": "node16",
    "target": "es2022"
  }
}
```

```ts
// ❌ import { sum } from "./util";        → 上記エラー
// ✅ import { sum } from "./util.js";     // .ts でも出力後の .js を書く
import { sum } from "./util.js";
```

## "Cannot find module or its type declarations" を tsc --traceResolution で根拠特定

型定義が見つからない系の8割は解決アルゴリズムの取り違え。推測せず `--traceResolution` で実際の探索ログを出す。

```bash
npx tsc --traceResolution --noEmit 2>&1 | grep -A3 "date-fns"
```

```text
Resolving with primary search path '.../node_modules/@types'.
File '.../node_modules/date-fns/package.json' exists - using "exports".
'package.json' has 'exports' field.   ← node10では読まれず失敗する
Module name 'date-fns', resolved to '.../index.d.ts'. SUCCESS
```

`exists - using "exports"` の行が出ない場合、`moduleResolution: "node"`（旧 `node10`）が `exports` フィールドを無視している。`"bundler"` か `"node16"` に上げれば解決する。

## Vite が "bundler" を推す一方 Node では "node16" が要る分岐

Vite 公式テンプレが `"bundler"` を採用するのは、拡張子省略を許しつつ `exports` を解決できるバンドラ前提の挙動に最も近いため。だが `"bundler"` は `tsc` で実行可能な JS を出さない（拡張子なし import がそのまま残る）。実行コードを `tsc` で出す部分だけ `"node16"` に分ける。

```jsonc
// tsconfig.json (アプリ側 / Viteがバンドル)
{ "compilerOptions": { "module": "esnext", "moduleResolution": "bundler" } }
```

```jsonc
// tsconfig.node.json (vite.config.ts や scripts/ をNode実行)
{ "compilerOptions": { "module": "node16", "moduleResolution": "node16" } }
```

## "is not under 'rootDir'" / exports 解決失敗の最小diff

`package.json` の `exports` が原因で `Package subpath './foo' is not defined by "exports"` が出るケース。`"bundler"` 以上に上げるのが最小修正。

```diff
 {
   "compilerOptions": {
-    "moduleResolution": "node",
+    "moduleResolution": "bundler",
+    "resolvePackageJsonExports": true,
     "module": "esnext"
   }
 }
```

## 30秒で自分の値を確定するフローチャート

```bash
# 判定スクリプト: 出力された行をそのまま採用
node -e '
const p=require("./package.json");
const node=process.argv.includes("--run");
if(p.type!=="module") console.log("→ \"node\" (CommonJS, 旧構成)");
else if(node) console.log("→ \"node16\" (.js拡張子必須)");
else console.log("→ \"bundler\" (Vite/webpackがバンドル)");
' -- "$@"
```

判定基準は3点のみ。①`"type":"module"` がない→`node`。②付いていて、その tsconfig が `tsc` で実行する JS（`vite.config.ts`・`scripts/`）を出す→`node16`。③それ以外（Vite がバンドルするアプリコード）→`bundler`。この分岐で本章の12エラーはすべて再現しなくなる。
