---
title: "jest設定でESM importがSyntaxErrorになる"
emoji: "⚙️"
type: "tech"
topics: ["JavaScript", "TypeScript", "設定"]
published: true
---

## 発生条件

- `package.json` に `"type": "module"` が設定されているか、`.mjs` 拡張子のファイルを使用している
- Jest をデフォルト設定のまま使用しており、`transform` の設定が `babel-jest` または `ts-jest` に向いていない
- テスト対象のモジュールや依存ライブラリが ESM 形式（`import`/`export` 構文）で書かれている

## 原因

以下のような設定でエラーが発生する。

```js
// jest.config.js（問題のある設定）
module.exports = {
  testEnvironment: 'node',
};
```

```json
// package.json
{
  "type": "module",
  "devDependencies": {
    "jest": "^29.0.0"
  }
}
```

Jest は内部的に CommonJS を前提として動作する。`package.json` に `"type": "module"` が指定されていると、プロジェクト内の `.js` ファイルはすべて ESM として扱われる。しかし Jest のデフォルト設定では `import` 文をそのまま実行しようとするため、Node.js の CJS ランタイム上で `SyntaxError: Cannot use import statement in a module` が発生する。

`transform` を指定していない場合、Jest はソースファイルを変換せずに Node.js に渡す。Node.js 側は `"type": "module"` の指示に従って ESM として解釈しようとするが、Jest の実行環境がそれに対応していないため構文エラーになる。

## 修正方法

**方法A: Babel で CJS にトランスパイルする（既存プロジェクト向け）**

```bash
npm install --save-dev babel-jest @babel/core @babel/preset-env
```

```js
// babel.config.js
module.exports = {
  presets: [
    ['@babel/preset-env', { targets: { node: 'current' } }],
  ],
};
```

```js
// jest.config.js
module.exports = {
  transform: {
    '^.+\\.m?js$': 'babel-jest',
  },
};
```

`@babel/preset-env` が `import`/`export` を CJS の `require`/`module.exports` に変換するため、Jest が正常に読み込める。

**方法B: Jest のネイティブ ESM サポートを使う**

```json
// package.json に追記
{
  "scripts": {
    "test": "node --experimental-vm-modules node_modules/.bin/jest"
  }
}
```

```js
// jest.config.js
export default {
  extensionsToTreatAsEsm: ['.ts'],
  testEnvironment: 'node',
};
```

`--experimental-vm-modules` フラグを付けて Jest を起動すると、ESM のまま実行できる。ただし `jest.config.js` 自体も ESM 形式（`export default`）にする必要がある。`package.json` の `"type": "module"` はそのまま維持できる。

TypeScript を使っている場合は `ts-jest` の `useESM: true` オプションと組み合わせる。

```js
// jest.config.js（TypeScript + ESM）
export default {
  extensionsToTreatAsEsm: ['.ts'],
  transform: {
    '^.+\\.ts$': ['ts-jest', { useESM: true }],
  },
};
```

どちらの方法も、設定変更後は `node_modules/.cache/jest` を削除してからテストを実行すると、キャッシュ起因の混乱を避けられる。

---

## JS/TS設定エラーをAIが30秒で根本診断
→ [診断キット](https://zenn.dev/itsuya_auto/books/jsconfig-diag-kit)
