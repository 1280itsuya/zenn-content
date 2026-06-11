---
title: "Error [ERR_REQUIRE_ESM]: require() of ES Module not supported"
emoji: "🐛"
type: "tech"
topics: ["Node.js", "JavaScript", "エラー解決", "npm"]
published: true
---

## 発生条件

- Node.js で CommonJS 形式（`require()`）のファイルから、ESM-only（`"type": "module"` のみで配布される）パッケージを読み込んだとき。
- パッケージが新しいメジャーバージョンで ESM 専用へ移行し、`require()` のままアップデートした直後。
- `package.json` に `"type": "module"` を付け忘れたプロジェクトで、ESM-only ライブラリを使おうとしたとき。

## 原因

`Error [ERR_REQUIRE_ESM]: require() of ES Module not supported` は、CommonJS 側の `require()` が ESM-only パッケージを読み込もうとして発生します。ESM-only パッケージは `module.exports` を持たないため、`require()` では解決できません。

```js
// index.js (CommonJS)
// node-fetch v3 や chalk v5 などは ESM-only
const fetch = require('node-fetch');

async function main() {
  const res = await fetch('https://example.com');
  console.log(res.status);
}

main();
```

この `require('node-fetch')` を実行した瞬間に `ERR_REQUIRE_ESM` で停止します。

## 修正方法

ファイル全体を ESM に切り替えられるなら、`package.json` に `"type": "module"` を追加して `import` 文を使います。

```js
// index.js (ESM) ※ package.json に "type": "module" を追加
import fetch from 'node-fetch';

const res = await fetch('https://example.com');
console.log(res.status);
```

CommonJS のまま据え置きたい場合は、`require()` を `await import()`（dynamic import）に置き換えます。これなら `package.json` を変更せず、ESM-only パッケージを読み込めます。

```js
// index.js (CommonJS のまま)
async function main() {
  const { default: fetch } = await import('node-fetch');

  const res = await fetch('https://example.com');
  console.log(res.status);
}

main();
```

要点は次の3つです。

- `require() of ES Module not supported` が出たパッケージは ESM-only なので、`require()` では読めない。
- プロジェクト全体を移行できるなら `"type": "module"` ＋ `import` が素直。
- 部分的な対応なら `await import()` を使い、`{ default: ... }` で本体を取り出す（トップレベルが関数の外なら `async` 関数で囲む）。

`require()` を残したまま、対象パッケージを古い CommonJS 対応バージョンに固定するのも回避策ですが、セキュリティ更新が止まるため、可能なら `import` 系への移行を推奨します。

## 似たエラーの解決記事

- [Error: Cannot find module 'express'](https://zenn.dev/articles/err-7b4f33b26a2d46)


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/aikit260611)- 原因を根本から理解するなら体系的な技術書（Amazon: [Node.js 実践入門](https://www.amazon.co.jp/s?k=Node.js%20%E5%AE%9F%E8%B7%B5%E5%85%A5%E9%96%80&tag=1280itsuya22-22)）
