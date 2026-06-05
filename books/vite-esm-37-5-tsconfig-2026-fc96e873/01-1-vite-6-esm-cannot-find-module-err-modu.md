---
title: "第1章 Vite 6+ESMで必ず出る『Cannot find module / ERR_MODULE_NOT_FOUND』Top7をdiffで即修正"
free: true
---

## この本の使い方=エラー全文で本文内検索→最小diffをコピペ

詰まったら解説を読むな。エディタやブラウザに出た**エラーメッセージを丸ごとコピー**し、`Ctrl+F`でこの本を全文検索しろ。各エラーには「全文→2行原因→tsconfigの最小diff」だけが並ぶ。読む順番は不要、ヒットした節だけ見て直す。本章は頻出7件、残り30件は章末索引にある。

```bash
# まず自分の環境を確定させる(diffが効く前提条件)
node -v   # v20.x を想定
npx vite --version   # vite/6.x を想定
cat tsconfig.json | grep -E "moduleResolution|verbatimModuleSyntax"
```

## `ERR_MODULE_NOT_FOUND`:拡張子`.js`付け忘れと`node10`残骸

```
Error [ERR_MODULE_NOT_FOUND]: Cannot find module '/app/src/util' imported from /app/src/index.js
```

原因はESMでは相対importに拡張子が必須なのに省略している、または`moduleResolution`が古い`node10`(旧`node`)のまま。直すdiffはこの2行。

```diff
// tsconfig.json
-    "moduleResolution": "node",
+    "moduleResolution": "bundler",
```
```diff
// src/index.ts — import側
-import { fmt } from "./util";
+import { fmt } from "./util.js";
```

`bundler`なら拡張子省略を許すが、Node直実行も併用するなら`.js`付与で両対応にする。

## `type: module`とCJS混在で出る`require is not defined`

```
ReferenceError: require is not defined in ES module scope, you are using require()
```

`package.json`に`"type": "module"`があるのにファイルが`require`/`__dirname`を使っている。`__dirname`はESMに存在しない。

```diff
// package.json
   "type": "module",
```
```ts
// src/path.ts — __dirname is not defined の解決
import { fileURLToPath } from "node:url";
import { dirname } from "node:path";
const __dirname = dirname(fileURLToPath(import.meta.url));
```

CJSのまま残したい設定ファイルは拡張子を`.cjs`に変える(例:`postcss.config.cjs`)。

## `paths`が解決されない:`vite-tsconfig-paths`を入れる

```
Failed to resolve import "@/lib/api" from "src/app.ts". Does the file exist?
```

tsconfigの`paths`はTypeScriptの型解決専用で、Viteの実行時バンドルには伝わらない。プラグインで橋渡しする。

```diff
// tsconfig.json
+    "baseUrl": ".",
+    "paths": { "@/*": ["src/*"] }
```
```ts
// vite.config.ts
import tsconfigPaths from "vite-tsconfig-paths";
export default { plugins: [tsconfigPaths()] };
```

## `verbatimModuleSyntax`の`import type`強制と`.ts`拡張子import

```
error TS1484: 'User' is a type and must be imported using a type-only import when 'verbatimModuleSyntax' is enabled.
An import path can only end with a '.ts' extension when 'allowImportingTsExtensions' is enabled.
```

前者は型を値importしている、後者は`.ts`拡張子importの許可フラグ欠落。両方tsconfigと書き方で潰す。

```diff
// tsconfig.json
+    "verbatimModuleSyntax": true,
+    "allowImportingTsExtensions": true,
+    "noEmit": true
```
```diff
// 型は type-only import に分離
-import { User } from "./types";
+import type { User } from "./types";
```

## Vite 6/Node 20で緑になるtsconfig.json完成形

ここまでの7エラーを同時に満たす配布用の完成形。新規プロジェクトはこれを丸ごと置けば本章の地雷を全て回避できる。

```jsonc
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "verbatimModuleSyntax": true,
    "allowImportingTsExtensions": true,
    "noEmit": true,
    "strict": true,
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  },
  "include": ["src", "vite.config.ts"]
}
```

この設定で`npx tsc --noEmit && npx vite build`が両方通れば第1章は完了だ。だが`tsc`は通るのに`vite build`だけ落ちる`Top-level await`系や、`esbuild`の`The injected path ... cannot be marked as external`など、設定が正しくても出る非対称エラーが残り30件ある。第2章以降は同じ「エラー全文→最小diff」形式でその30件を索引化している。今出ているメッセージを`Ctrl+F`で次章の索引に投げれば、解説を読まずに直せる。
