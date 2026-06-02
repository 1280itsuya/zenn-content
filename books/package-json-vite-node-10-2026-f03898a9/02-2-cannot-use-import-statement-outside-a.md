---
title: "第2章: 『Cannot use import statement outside a module』— type:module欠落とESM/CJS混在を差分1行で解決"
free: false
---

## `type:module`未設定で出る3つのエラーを1秒で見分ける

結論から書く。`Cannot use import statement outside a module` / `require is not defined` / `__dirname is not defined` の3つは、すべて `package.json` の `"type"` フィールド1行で発生源が確定する。Node 20.11 で実際に再現させると、エラー文字列とビルド環境は次の対応になる。

```bash
# Vite 5.4 + Node 20.11 で再現
$ node vite.config.js
# type 未設定 → SyntaxError: Cannot use import statement outside a module
# "type":"module" 追加後に古い require が残存 → ReferenceError: require is not defined
# 同上で __dirname 参照 → ReferenceError: __dirname is not defined
```

3つは独立した不具合ではなく、`"type":"module"` を入れた瞬間に連鎖して切り替わる。次節から差分1行単位で潰す。

## `"type":"module"` 追加でimport文エラーを消すbefore/after差分

`Cannot use import statement outside a module` は、`.js` 拡張子のファイルが CommonJS として解釈されているだけ。`package.json` に1行足して解決する。

```diff
 {
   "name": "my-vite-app",
+  "type": "module",
   "scripts": {
     "dev": "vite"
   }
 }
```

この1行で `vite.config.js` 内の `import { defineConfig } from 'vite'` が通る。`tsconfig.json` の `"module"` 設定や Vite 本体のバージョンは触らない。変更は `package.json` の1行だけで足りる。

## `require is not defined` を `createRequire` で復旧する2行修正

`"type":"module"` を入れた直後、CJS のまま残った `require('./config.cjs')` が `require is not defined in ES module scope` を吐く。ESM には `require` が存在しないため、`module` から復元する。

```js
// vite.config.js (ESM)
import { createRequire } from 'node:module'
const require = createRequire(import.meta.url)

const legacy = require('./legacy.config.cjs') // CJSパッケージを温存したまま読める
```

外部CJSパッケージを一括で `import` に書き換えられない移行期に有効。書き換え対象を `createRequire` の2行に閉じ込められる。

## `__dirname is not defined` を `import.meta.url` で再現する

ESM では `__dirname` と `__filename` がグローバルから消える。Vite の `resolve.alias` で絶対パスを組むコードがここで落ちる。`fileURLToPath` で等価の値を再構成する。

```js
import { fileURLToPath } from 'node:url'
import { dirname, resolve } from 'node:path'

const __filename = fileURLToPath(import.meta.url)
const __dirname = dirname(__filename)

export default {
  resolve: { alias: { '@': resolve(__dirname, 'src') } }
}
```

`process.cwd()` で代用するとサブディレクトリ実行時にエイリアスがずれる。`import.meta.url` 基準が唯一安全。

## `.mjs`/`.cjs` への改名で巻き込まれる周辺ファイル一覧

`"type":"module"` の影響は `vite.config.js` だけでは終わらない。拡張子なし設定ファイルが連鎖的に ESM 判定される。改名で個別に CJS へ逃がす判断基準を表にする。

```text
postcss.config.js   → module.exports 残すなら postcss.config.cjs
tailwind.config.js  → 同上 tailwind.config.cjs
.eslintrc.js        → eslint v9 で eslint.config.mjs へ
vite.config.js      → import 統一なら .js のまま / 混在なら .mjs
```

判断は単純で、中身が `export default` なら `.js`(or `.mjs`)、`module.exports` なら `.cjs`。拡張子とエクスポート構文を一致させた瞬間に判定の曖昧さが消える。

## 7回ビルドを失敗させて到達した正解形 `package.json`

混在解消を急いで全ファイルを一度に ESM 化し、`postcss.config.js` の `module.exports` を見落として7回ビルドを落とした。最終的に通った構成は「`package.json` は ESM、PostCSS だけ `.cjs` に隔離」だった。丸ごと掲載する。

```json
{
  "name": "my-vite-app",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build"
  },
  "devDependencies": {
    "vite": "^5.4.0",
    "postcss": "^8.4.0",
    "tailwindcss": "^3.4.0"
  }
}
```

これに `postcss.config.cjs`(module.exports 形式)を併置すれば、`npm run build` が ESM/CJS 混在のまま1回で通る。第3章では同じ差分手法を `vite: command not found` 系のパス解決エラーへ展開する。
