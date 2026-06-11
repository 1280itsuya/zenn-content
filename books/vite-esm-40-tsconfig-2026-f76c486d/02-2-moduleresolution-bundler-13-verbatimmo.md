---
title: "第2章 moduleResolution:\"bundler\"移行で壊れる13パターンとverbatimModuleSyntax対応diff"
free: false
---

## 結論：bundler移行のエラー13個は「tsconfig 2行」と「import文の書き換え」に集約される

`moduleResolution:"bundler"` で出る13パターンは原因が3系統しかない。allowImportingTsExtensions系・verbatimModuleSyntax系・isolatedModules系だ。逆引きで使えるよう、エラー全文→diffの順で並べる。

```diff
// tsconfig.json — bundler移行の最小セット
   "moduleResolution": "node",
+  "moduleResolution": "bundler",
+  "allowImportingTsExtensions": true,
+  "verbatimModuleSyntax": true,
+  "isolatedModules": true,
   "noEmit": true
```

## TS1484: type-only importでverbatimModuleSyntaxが落ちる修正diff

`verbatimModuleSyntax:true` で最初に出るのがこれ。型を値と同じimport文で読むと即エラーになる。

```
error TS1484: 'User' is a type and must be imported using a type-only import
when 'verbatimModuleSyntax' is enabled.
```

修正は `type` 修飾子の付与だけ。値と型が混在する行は分割する。

```diff
-import { fetchUser, User } from "./api";
+import { fetchUser } from "./api";
+import type { User } from "./api";
```

往復ロス: VSCodeのQuick Fix「Add 'type' to import」を一括適用せず手で直して**約40分**溶かした。`eslint-plugin-import` の `consistent-type-imports` を入れると自動化できる。

## TS5097: allowImportingTsExtensionsで.ts拡張子が必須化されるdiff

bundler環境では拡張子なしimportがエラーになる場合がある。Viteは拡張子付きを解決できるため、明示する方が安全だ。

```
error TS5097: An import path can only end with a '.ts' extension
when 'allowImportingTsExtensions' is enabled.
```

```diff
-import { config } from "./config";
+import { config } from "./config.ts";
```

注意: `allowImportingTsExtensions:true` は `noEmit:true` か `emitDeclarationOnly:true` とセットでないとTS5096で弾かれる。Viteは型を出力しないので `noEmit` を必ず立てる。

## TS1286: const enumがisolatedModulesで壊れる、union型への置換diff

`isolatedModules:true` 下では `const enum` がトランスパイル単位を跨げず落ちる。Viteのesbuildは1ファイル単位で変換するためだ。

```
error TS1286: A 'const' enum member cannot be accessed
using an inline reference when 'isolatedModules' is enabled.
```

`const enum` を `as const` オブジェクトに置換するとランタイムも軽くなる。

```diff
-export const enum Role { Admin, Guest }
+export const Role = { Admin: 0, Guest: 1 } as const;
+export type Role = (typeof Role)[keyof typeof Role];
```

## node か bundler か：再エクスポート崩壊を防ぐ判断フロー

CommonJSパッケージを `export *` で再エクスポートしているライブラリを抱える場合、bundlerに切ると `exports` が解決されず崩れることがある。判断はビルドツールで割り切る。

```bash
# Vite / esbuild / Webpack5 を使う → bundler
# tsc単体で.jsを出力する / Node直実行 → node16 or nodenext
test -f vite.config.ts && echo "use: bundler" || echo "use: nodenext"
```

判断ミスでnode→bundler→nodenextを往復し、`package.json` の `"type":"module"` 欠落も重なって**実2時間**を失った。先に上記1行で確定させれば、この章のdiffを上から貼るだけで13パターンは塞げる。
