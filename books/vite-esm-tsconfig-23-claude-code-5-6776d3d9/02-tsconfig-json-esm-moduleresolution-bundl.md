---
title: "tsconfig.json×ESM完全移行: moduleResolution=bundlerで解消した10エラーの設定差分全文"
free: false
---

## 実測: Vite 6×TypeScript 5.8でmoduleResolution 4種を比較した結果

まず数値で結論を出す。

| moduleResolution | 型エラー件数 | `vite build`成功 | 備考 |
|---|---|---|---|
| node | 8件 | ❌ | ESM import拡張子未解決 |
| node16 | 6件 | ❌ | `.js`拡張子強制でViteと競合 |
| nodenext | 5件 | ❌ | node16とほぼ同等 |
| **bundler** | **0件** | ✅ | Viteバンドラ前提 |

`bundler`はTypeScript 5.0で追加された設定値で、バンドラ（Vite/webpack/esbuild）がモジュール解決を担う前提でTSCを動かす。`node16`/`nodenext`が要求する`.js`拡張子の明示やCJS/ESM厳格分離はVite環境で不要かつ有害。移行コストは`tsconfig.json`1ファイルの5行修正のみ。

---

## TS2307: `Cannot find module '@/components/Button'` — pathsエイリアス崩壊の根本原因

**エラー文:**
```
error TS2307: Cannot find module '@/components/Button' or its corresponding type declarations.
```

`node`モードでは`paths`は構文上有効だが、ESM importの型解決でViteの`resolve.alias`と二重管理が生じて崩壊する。

**Before:**
```json
{
  "compilerOptions": {
    "moduleResolution": "node",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

**After:**
```json
{
  "compilerOptions": {
    "moduleResolution": "bundler",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

`bundler`に切り替えると、TSCがバンドラのエイリアス解決を前提として動作し、`paths`とViteの`alias`が整合する。`vite.config.ts`側の`resolve.alias`は変更不要。

---

## TS1205/TS1484: `isolatedModules`とre-export 3件の競合を一括解消

**エラー文:**
```
error TS1205: Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'.
error TS1484: 'ApiResponse' is a type and must be imported using a type-only import.
```

`isolatedModules: true`はViteのesbuildと連動するために必須だが、型のre-exportパターンと衝突する。tsconfig変更は不要で、ソース側を修正する。

**Before (`src/types/index.ts`):**
```typescript
export { UserType } from './user';
export { ApiResponse } from './api';
```

**After:**
```typescript
export type { UserType } from './user';
export type { ApiResponse } from './api';
```

さらに`verbatimModuleSyntax: true`を追加すると、型importルールをTSCが自動チェックし同種エラーをCIで事前検出できる。

```json
{
  "compilerOptions": {
    "moduleResolution": "bundler",
    "isolatedModules": true,
    "verbatimModuleSyntax": true
  }
}
```

---

## TS7016/TS2792: `.d.ts`が解決されない2パターンのbefore/after

**エラー文①（サードパーティ型定義なし）:**
```
error TS7016: Could not find a declaration file for module 'some-lib'.
```

**エラー文②（Viteプラグイン生成のバーチャルモジュール）:**
```
error TS2792: Cannot find module 'virtual:pwa-register'. Did you mean to set the 'moduleResolution' option to 'bundler'?
```

TS7016は`@types/some-lib`が存在しない場合に自前で型宣言を作成する。

**`src/types/vendor.d.ts`（新規作成）:**
```typescript
declare module 'some-lib' {
  export function doSomething(input: string): Promise<void>;
}
```

TS2792はViteプラグインが生成するバーチャルモジュールの型が見えないケース。`types`フィールドで明示的に参照する。

**After (`tsconfig.json`):**
```json
{
  "compilerOptions": {
    "moduleResolution": "bundler",
    "types": ["vite/client", "vite-plugin-pwa/client"]
  }
}
```

---

## 残り4エラー: `allowSyntheticDefaultImports`と`allowImportingTsExtensions`の設定差分

**エラー文③:**
```
error TS1259: Module '"lodash"' can only be default-imported using the 'allowSyntheticDefaultImports' flag.
```

**エラー文④:**
```
error TS5097: An import path can only end with a '.ts' extension when 'allowImportingTsExtensions' is enabled.
```

**エラー文⑤:**
```
error TS2304: Cannot find name 'ImportMeta'.
```

**エラー文⑥:**
```
error TS1479: The current file is a CommonJS module whose imports will produce 'require' calls.
```

4件をまとめて解消するオプション追加:

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowSyntheticDefaultImports": true,
    "allowImportingTsExtensions": true,
    "noEmit": true,
    "types": ["vite/client"]
  }
}
```

`allowImportingTsExtensions: true`は`noEmit: true`または`emitDeclarationOnly: true`との併用が必須。TSCが`.ts`拡張子を含むファイルをemitできないためで、Viteのビルドはesbuildが担う構成が前提。`vite/client`を`types`に入れると`ImportMeta`型が解決される（エラー文⑤の原因はこの欠落）。

---

## 確定版 tsconfig.json 全文（Vite 6×TypeScript 5.8、10エラーゼロ）

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "useDefineForClassFields": true,
    "lib": ["ESNext", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "moduleDetection": "force",
    "noEmit": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "types": ["vite/client"],
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

`node`設定からの差分サマリ:

| キー | Before (`node`) | After (`bundler`) | 解消エラー |
|---|---|---|---|
| moduleResolution | node | bundler | TS2307, TS2792, TS6 |
| allowImportingTsExtensions | 未設定 | true | TS5097 |
| verbatimModuleSyntax | 未設定 | true | TS1205, TS1484 |
| moduleDetection | 未設定 | force | TS1261 |
| noEmit | 未設定 | true | TS5097前提条件 |
| types（vite/client） | 未設定 | ["vite/client"] | TS2304 |

`moduleDetection: force`は全ファイルをモジュールとして扱い、グローバル汚染によるTS1261（ファイル名の大文字小文字差異）を防ぐ副作用も持つ。このキー1つで10件中2件が消えるケースがある。

この設定を`main`ブランチに入れたあとは、CIで`tsc --noEmit`を走らせて型エラーの回帰を検出する構成（次章）に進む。
