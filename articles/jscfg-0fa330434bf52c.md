---
title: "webpack module not foundの原因別修正"
emoji: "⚙️"
type: "tech"
topics: ["JavaScript", "TypeScript", "設定"]
published: true
---

## 発生条件

- `resolve.alias` や `resolve.modules` を誤って設定した場合
- 絶対パスでインポートしているのに `tsconfig.json` の `paths` を webpack 側に伝えていない場合
- パッケージのエントリポイントが `exports` フィールドで制限されているのに古いフィールドを参照している場合

## 原因

以下のような webpack 設定で `Module not found: Error: Can't resolve '@components/Button'` が発生する。

```js
// webpack.config.js（壊れている例）
module.exports = {
  resolve: {
    alias: {
      '@components': './src/components', // 相対パスを指定している
    },
  },
};
```

`resolve.alias` に指定するパスは**絶対パスでなければならない**。相対パスを書くと webpack はそれをプロジェクトルートからの相対パスとして解釈するのではなく、そのまま文字列として扱うため、ファイルを見つけられない。

`tsconfig.json` に `paths` を書いていても、webpack はそれを読まないため別途 alias の設定が必要になる。

```json
// tsconfig.json（paths あり）
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@components/*": ["src/components/*"]
    }
  }
}
```

この設定は TypeScript コンパイラが型チェックに使うものであり、webpack のモジュール解決には影響しない。両者は独立して動作する。

## 修正方法

`resolve.alias` には `path.resolve` を使って絶対パスを指定する。

```js
// webpack.config.js（正しい例）
const path = require('path');

module.exports = {
  resolve: {
    alias: {
      '@components': path.resolve(__dirname, 'src/components'),
    },
  },
};
```

`path.resolve(__dirname, 'src/components')` によって、設定ファイルの場所を基点とした絶対パスが生成される。これで webpack が正しくファイルを探せるようになる。

`tsconfig.json` の `paths` との整合性を自動で取りたい場合は `tsconfig-paths-webpack-plugin` を使う方法もある。

```js
// webpack.config.js（tsconfig-paths-webpack-plugin を使う例）
const path = require('path');
const TsconfigPathsPlugin = require('tsconfig-paths-webpack-plugin');

module.exports = {
  resolve: {
    plugins: [
      new TsconfigPathsPlugin({
        configFile: path.resolve(__dirname, 'tsconfig.json'),
      }),
    ],
  },
};
```

このプラグインを使うと `tsconfig.json` の `paths` を webpack が自動的に読み込むため、エイリアスを二重管理せずに済む。ただしプラグインを導入する場合でも `tsconfig.json` 側の `baseUrl` が正しく設定されていることが前提となる。

`exports` フィールドによる問題はパッケージ側の制約なので、該当パッケージのドキュメントを確認してサブパスを正しく指定するか、`resolve.extensionAlias` や `resolve.conditionNames` で条件を調整する。