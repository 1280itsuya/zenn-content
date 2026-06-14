---
title: "第2章：tsconfig.json 4設定値でESM/CJS衝突を8分で終わらせる——module/moduleResolution/targetの正解値一覧"
free: false
---

## 「型は通るのに実行時クラッシュ」の正体：TypeScriptの2パスが独立している

TypeScriptコンパイラは2つの独立したパスで動作する。

1. **型チェックパス**: `.ts`を解析して型エラーを検出（`tsc --noEmit`の担当）
2. **モジュール解決パス**: `import`/`require`をJSに変換し、Node.jsが実際に読み込めるパスを決定

`tsc`が「エラーなし」を返しても、生成した`.js`が`require`を使っているのに`.mjs`として実行されればNode.jsはクラッシュする。`module`設定は型チェックにほぼ影響しない。影響するのはモジュール解決と出力構文だ。

```
[型チェックパス]  → "types" / "lib" / "strict" が支配
[モジュール解決]  → "module" / "moduleResolution" が支配
         ↑ 2パスが独立しているため「型OK・実行NG」が発生する
```

## Node.js 18/20/22 × module/target の正解マトリクス

| Node.js | `package.json "type"` | module | moduleResolution | target |
|---------|----------------------|--------|-----------------|--------|
| 18 | (なし) | CommonJS | node | ES2022 |
| 18 | "module" | NodeNext | NodeNext | ES2022 |
| 20 | (なし) | CommonJS | node | ES2022 |
| 20 | "module" | NodeNext | NodeNext | ES2023 |
| 22 | "module" | NodeNext | NodeNext | ES2024 |
| 22 | "module" + バンドラー | ESNext | bundler | ES2023 |

**`"module": "ESNext"`はNode.jsで直接実行するコードに使わない**。Webpack/Vite/esbuildがバンドルする前提の設定だ。`ts-node`や`tsx`でスクリプトを直接実行するなら`NodeNext`一択。

```jsonc
// ❌ Node.js 20 + tsx で直接実行 → クラッシュする組み合わせ
{
  "compilerOptions": {
    "module": "ESNext",         // バンドラー向け設定
    "moduleResolution": "node"  // ESNext と不整合
  }
}
```

## moduleResolution 3値の使い分け：Claude SDK・Playwright・zenn-cli が要求する値

```
@anthropic-ai/sdk 0.27+:
  → package.json に "exports" フィールドあり
  → bundler または NodeNext が必要
  → "node" を指定すると型解決が壊れ Cannot find module が多発

@playwright/test 1.44+:
  → CJS/ESM 両対応だが playwright.config.ts は
    ts-node が CJS で実行するため CommonJS + node が安全
  → ESM プロジェクトでは playwright.config.ts を .mts にするか
    TS_NODE_PROJECT で tsconfig を分離する

zenn-cli 0.1.157+:
  → 内部が CJS で動作
  → 同一リポジトリに ESM の tsconfig.json を置くと
    zenn-cli が読む TS ファイルと衝突する場合がある
  → tsconfig.zenn.json を分離するか zenn-cli は TS 経由せず運用
```

```bash
# 実際に解決されている設定値を確認（extends 多段継承込み）
npx tsc --showConfig | grep -E '"module"|"moduleResolution"|"target"'
```

「設定したはずなのに効かない」の多くはこれで原因が判明する。

## 5パターンの tsconfig 差分：コピペで即解決

### パターン1：Claude SDK + Node.js 20（ESMスクリプト）

```diff
 {
   "compilerOptions": {
-    "module": "CommonJS",
-    "moduleResolution": "node",
+    "module": "NodeNext",
+    "moduleResolution": "NodeNext",
     "target": "ES2023",
     "outDir": "./dist",
     "strict": true
   }
 }
```

```jsonc
// package.json に追加必須
{ "type": "module" }
```

### パターン2：Playwright + TypeScript（CJS で安定稼働）

```diff
 {
   "compilerOptions": {
-    "module": "ESNext",
-    "moduleResolution": "bundler",
+    "module": "CommonJS",
+    "moduleResolution": "node",
     "target": "ES2022",
     "strict": true
   }
 }
```

### パターン3：Vite + React（バンドラー経由）

```diff
 {
   "compilerOptions": {
-    "module": "NodeNext",
-    "moduleResolution": "NodeNext",
+    "module": "ESNext",
+    "moduleResolution": "bundler",
     "target": "ES2020",
     "jsx": "react-jsx",
     "strict": true
   }
 }
```

### パターン4：Claude SDK と Playwright が同一リポジトリに共存

```
├── tsconfig.json               ← Claude SDK向け（NodeNext）
├── tsconfig.playwright.json    ← Playwright向け（CommonJS）
└── playwright.config.ts
```

```jsonc
// tsconfig.playwright.json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "module": "CommonJS",
    "moduleResolution": "node"
  },
  "include": ["playwright.config.ts", "tests/**/*.ts"]
}
```

```bash
TS_NODE_PROJECT=tsconfig.playwright.json npx playwright test
```

### パターン5：Node.js 18 レガシー + @anthropic-ai/sdk（型エラー回避）

```diff
 {
   "compilerOptions": {
     "module": "CommonJS",
-    "moduleResolution": "node",
+    "moduleResolution": "node16",
     "target": "ES2022",
-    "esModuleInterop": false,
+    "esModuleInterop": true,
     "strict": true
   }
 }
```

`@anthropic-ai/sdk`は`"exports"`フィールドを持つため`"moduleResolution": "node"`では型が解決されず`Cannot find module`が出る。`node16`以上にすると`exports`フィールドを読むようになり解消する。

## エラーメッセージ→設定変更の一対一対応表（診断8分）

```
Cannot find module 'xxx' or its corresponding type declarations
  → moduleResolution を node16 または NodeNext に変更

require() of ES module ... not supported
  → module を NodeNext に変更
    または package.json から "type":"module" を削除

The current file is a CommonJS module whose imports will produce 'require' calls
  → .ts ファイルを .mts にリネーム
    または package.json に "type":"module" を追加

An import path cannot end with a '.ts' extension
  → import "./foo.js" に変更（.ts ではなく出力後の .js を書く）

SyntaxError: Cannot use import statement in a module
  → package.json に "type":"module" を追加
    または出力拡張子を .mjs に変更
```

エラーメッセージを上の表に照合すれば、原因は1行に絞れる。設定変更は差分コピペで完結する。
