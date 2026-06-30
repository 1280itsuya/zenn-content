---
title: "tsconfig.json paths alias解決エラー"
emoji: "⚙️"
type: "tech"
topics: ["JavaScript", "TypeScript", "設定"]
published: true
---

## 発生条件

- `tsconfig.json` の `paths` でエイリアスを定義しているが、バンドラー（Vite・webpack・Jest など）側に同等の設定を追加していない
- `baseUrl` を省略、または `paths` と整合しないディレクトリを指定している
- モノレポ構成で `tsconfig.json` が複数存在し、実行時に参照されるファイルと `paths` を定義したファイルが食い違っている

## 原因

TypeScript の `paths` はあくまで**型チェック専用の解決ルール**であり、実際のファイル解決はバンドラーや Node.js のモジュールローダーが担う。そのため、`tsconfig.json` だけに書いても実行時・ビルド時には無視される。

よくある壊れた設定の例:

```jsonc
// tsconfig.json（問題のある設定）
{
  "compilerOptions": {
    // baseUrl が未指定、または paths のルートと合っていない
    "paths": {
      "@components/*": ["./src/components/*"]
    }
  }
}
```

`baseUrl` が省略されると `paths` のルートが不定になり、TypeScript 自身も解決できない。加えて、Vite や webpack はこの定義を読まないため、ビルド時に `Cannot find module '@components/Button'` などのエラーが出る。

## 修正方法

**Step 1: `tsconfig.json` を正しく設定する**

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",          // プロジェクトルートを基準に
    "paths": {
      "@components/*": ["src/components/*"],
      "@utils/*": ["src/utils/*"]
    }
  }
}
```

`baseUrl` は `paths` のルートになるため、必ずセットで指定する。

**Step 2: バンドラー側にも同じエイリアスを追加する**

Vite の場合:

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import path from 'path'

export default defineConfig({
  resolve: {
    alias: {
      '@components': path.resolve(__dirname, 'src/components'),
      '@utils': path.resolve(__dirname, 'src/utils'),
    },
  },
})
```

webpack の場合:

```js
// webpack.config.js
const path = require('path')

module.exports = {
  resolve: {
    alias: {
      '@components': path.resolve(__dirname, 'src/components'),
      '@utils': path.resolve(__dirname, 'src/utils'),
    },
  },
}
```

**Step 3: Jest を使う場合は `moduleNameMapper` も設定する**

```jsonc
// jest.config.json
{
  "moduleNameMapper": {
    "^@components/(.*)$": "<rootDir>/src/components/$1",
    "^@utils/(.*)$": "<rootDir>/src/utils/$1"
  }
}
```

要点は「`tsconfig.json` の `paths` は型チェックにしか効かない」という前提を踏まえ、**利用するすべてのツールに同じエイリアスを明示的に設定する**こと。`vite-tsconfig-paths` などのプラグインを使うと `tsconfig.json` の定義をバンドラーへ自動的に連携できるため、設定の二重管理を避けたい場合は導入を検討するとよい。