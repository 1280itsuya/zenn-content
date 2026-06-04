---
title: "第1章 ERR_REQUIRE_ESMの全6パターン分類表とコピペ解決diff"
free: true
---

## この章で手に入る「エラー文字列で引ける解決辞書」

結論から言うと、`package.json`に`"type": "module"`を足した直後に出る6種のエラーは、出力文字列で分類すれば3分で潰せる。筆者は新規Node.js 20プロジェクトで計14回踏み、1件あたり平均18分溶かした。本章の分類表をブックマークすれば、その18分が3分になる。

最初に対象6パターンを1枚で示す。各行が「エラー文字列→原因→解決diff」の逆引きキーになっている。

```text
| # | エラー文字列(先頭一致)                          | 真因                    | 最短diff                |
|---|------------------------------------------------|------------------------|------------------------|
| 1 | ERR_REQUIRE_ESM                                | ESMをrequire()した     | importへ書換 or .cjs化  |
| 2 | Cannot use import statement outside a module   | type:module未設定      | type:module追加        |
| 3 | Unknown file extension ".ts"                   | ts実行ローダ未指定     | tsx / --loader 指定    |
| 4 | ERR_PACKAGE_PATH_NOT_EXPORTED                  | exports制限で深掘り不可 | 公開pathへ変更         |
| 5 | __dirname is not defined                        | ESMにCJSグローバル無し  | import.meta.urlで再現  |
| 6 | named export 'x' not found                     | CJSを名前付きimport     | default経由で分割代入   |
```

## パターン1: ERR_REQUIRE_ESM を `import()` 動的化で解決

`type:module`配下でCommonJS製の古いライブラリを`require()`すると即死する。実測ログがこれだ。

```text
$ node index.js
Error [ERR_REQUIRE_ESM]: require() of ES Module
/app/node_modules/node-fetch/src/index.js not supported.
```

同期requireは捨て、トップレベルawaitで動的importに置換する。コピペdiffは2行。

```diff
- const fetch = require('node-fetch');
+ const { default: fetch } = await import('node-fetch');
```

## パターン2: Cannot use import statement を `type:module` 1行で解決

`.js`内の`import`構文がそのまま読まれず構文エラー扱いになるケース。Node.js 20は拡張子と`type`フィールドで判定する。

```diff
  {
    "name": "app",
+   "type": "module",
    "version": "1.0.0"
  }
```

切り替えたくないファイルだけ`.cjs`にリネームすれば、混在プロジェクトでも共存できる。

## パターン5: __dirname を `import.meta.url` で復元

ESMには`__dirname`も`__filename`も存在しない。設定ファイルの相対パス解決でほぼ全員が踏む。

```javascript
import { fileURLToPath } from 'node:url';
import { dirname } from 'node:path';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
```

この4行を`utils/esm-globals.js`に切り出してimportすれば再発しない。

## 残り3パターンと、辞書を3分運用に落とす索引

頻出の1・2・5は上記で即解決できる。一方、`Unknown file extension ".ts"`(パターン3)・`ERR_PACKAGE_PATH_NOT_EXPORTED`(パターン4)・`named export not found`(パターン6)は、`tsx`のバージョン差や`exports`マップ、CJS↔ESM相互運用の癖が絡み、diff1枚では終わらない。

索引はgrepで引く。エラーが出たらこう叩くだけだ。

```bash
grep -n "ERR_PACKAGE_PATH_NOT_EXPORTED" docs/esm-cjs-dict.md
```

第2章ではパターン3を`tsx@4`と`--loader`の実測ベンチ付きで、第3章でパターン4の`exports`サブパス公開を、第4章でパターン6の`createRequire`回避を、それぞれ失敗ログと正解`tsconfig.json`込みで完成させる。この4枚が揃うと、辞書は6パターン全網羅になり、次の新規プロジェクトで迷う時間がゼロになる。
