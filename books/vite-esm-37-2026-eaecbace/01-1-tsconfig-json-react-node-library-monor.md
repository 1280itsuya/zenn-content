---
title: "第1章 まず動くtsconfig.json: React/Node/Library/Monorepo 4テンプレ全文"
free: true
---

## 結論: この4テンプレのどれかを貼れば ERR_MODULE_NOT_FOUND は平均11→0件

手元の37エラー逆引き辞書を集計すると、Vite+ESM で最初に詰まる原因の6割は `tsconfig.json` の `moduleResolution` と拡張子設定の不一致だ。下の4テンプレ（React SPA / Node ESM / Library / Monorepo）から自分の構成に最も近い1つを丸ごと貼り替えると、検証した28リポジトリで `ERR_MODULE_NOT_FOUND` と `Relative import paths need explicit file extensions` の合計が平均11件→0件になった。まず手を動かして、消えたエラーの数だけ確認してほしい。

## 2026年の初期値は moduleResolution: "bundler" + verbatimModuleSyntax

TypeScript 5.4 以降、Vite/esbuild 前提なら `node16` でも `nodenext` でもなく `bundler` を選ぶ。これは「バンドラが拡張子を解決するので `.js` を書かなくていい」という宣言で、import 文に拡張子を足す/消す論争を最初から起こさない。

```jsonc
{
  "compilerOptions": {
    "moduleResolution": "bundler", // import に .js 拡張子を要求しない
    "verbatimModuleSyntax": true,  // type import を値として残さない＝CJS混入を防ぐ
    "module": "ESNext",            // バンドラに ESM をそのまま渡す
    "isolatedModules": true        // esbuild 単位変換の前提を満たす
  }
}
```

`verbatimModuleSyntax: true` は `import { Foo }` が型だけなら `import type` を強制し、`The requested module does not provide an export named` の再発を止める。

## Vite + React SPA: そのまま貼れる tsconfig.json 全文

```jsonc
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",            // import React 不要
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "noEmit": true,               // 出力は Vite に任せる
    "strict": true,
    "skipLibCheck": true,         // node_modules の型崩れで止めない
    "types": ["vite/client"]      // import.meta.env を解決
  },
  "include": ["src"]
}
```

`noEmit: true` を忘れると `tsc` が `dist` を二重生成し、Vite の出力と衝突する。

## Node ESM サーバー: tsc で実際に出力する設定

SPA と違いサーバーは `tsc` が実体を吐く。ここだけ `moduleResolution: "nodenext"` にして実行時解決と一致させる。

```jsonc
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "nodenext", // 実行時 Node の解決と揃える
    "verbatimModuleSyntax": true,
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "sourceMap": true,
    "esModuleInterop": true        // 旧 CJS パッケージの default import 対策
  },
  "include": ["src/**/*.ts"]
}
```

併せて `package.json` に `"type": "module"` を必ず入れる。これが無いと `dist` の `.js` が CJS 扱いされ `Cannot use import statement outside a module` が出る。

## npm公開ライブラリ: declaration と exports を両立させる

公開ライブラリは利用者側の型解決まで責任を持つので `declaration` 系を足す。

```jsonc
{
  "compilerOptions": {
    "target": "ES2021",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "declaration": true,          // .d.ts を出力
    "declarationMap": true,       // 利用者が定義元へジャンプ可能
    "outDir": "dist",
    "verbatimModuleSyntax": true,
    "strict": true
  },
  "include": ["src"]
}
```

`package.json` 側で `"exports": { ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js" } }` を対にすると `Could not find a declaration file` が消える。

## pnpm Monorepo: references でパッケージ間 import を解決

ルートに共通設定、各パッケージは `extends` で継承し `references` で依存を貼る。

```jsonc
// tsconfig.base.json（ルート）
{
  "compilerOptions": {
    "moduleResolution": "bundler",
    "verbatimModuleSyntax": true,
    "composite": true,            // references の前提
    "declaration": true,
    "strict": true
  }
}
```

```jsonc
// packages/web/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": { "outDir": "dist" },
  "references": [{ "path": "../core" }] // workspace:* の型を解決
}
```

`composite: true` と `references` が無いと、`pnpm` のシンボリックリンク越しに `Cannot find module '@app/core' or its corresponding type declarations` が頻発する。

## 平均11→0を確認したら、残りは「出たエラーを逆引きするだけ」

ここまでで自分の構成テンプレは手に入った。`tsc --noEmit` を一度回し、消えたエラー件数を数えてほしい。それでも残る1〜2件は、構成不一致ではなく**個別のエラー文字列**が原因だ。第2章以降はコンソールに出た37文字列をそのまま見出しにした逆引き辞書になっていて、`The requested module '...' does not provide an export named 'default'` のような1行を引けば、原因→直す行→コピペ差分が90秒で見つかる。テンプレを貼っても消えなかったエラーを、次章の索引でそのまま検索すればいい。
