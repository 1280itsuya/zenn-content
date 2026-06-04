---
title: "第3章 .mjs/.cjs/.ts拡張子とtsconfig module設定の組合せ表"
free: false
---

## 拡張子6種×type:moduleの真理値表（Node.js 20の解釈ルール）

Node.js 20 が各ファイルを ESM/CJS どちらで読むかは、拡張子と直近 `package.json` の `type` だけで決まる。tsconfig は一切関与しない。

```
拡張子   type:"module"   type:"commonjs"(既定)
.js      ESM             CJS
.mjs     ESM             ESM   ← 常にESM
.cjs     CJS             CJS   ← 常にCJS
```

`.mjs`/`.cjs` は `type` を無視して固定される。判定を確認するワンライナー:

```bash
node --input-type=module -e "console.log(import.meta.url)"   # ESMなら成功
node -e "console.log(typeof require)"                        # CJSなら function
```

## module:NodeNext と ESNext の tsc 出力diff（実測）

同一ソース `import { readFile } from "node:fs/promises"` を tsc 5.4 で変換すると、`module` の値で出力が割れる。

```diff
# module: "CommonJS"
-const promises_1 = require("node:fs/promises");
# module: "NodeNext" (.ts→.js, 直近 type:"module")
+import { readFile } from "node:fs/promises";
```

`NodeNext` は出力ファイルの拡張子と `type` から ESM/CJS を逐一判定する。`ESNext` は常に ESM 出力で、CJS 混在プロジェクトでは `require` 側から読めず `ERR_REQUIRE_ESM` を誘発する。Node 実行なら `NodeNext`、bundler 前提なら `ESNext`+`moduleResolution:"bundler"` が安全側。

## ERR_MODULE_NOT_FOUND の原因は import の .js 付け忘れ

NodeNext の最大の罠。実行ログに次が出たら拡張子漏れを疑う。

```
Error [ERR_MODULE_NOT_FOUND]: Cannot find module '/app/util' imported from /app/index.js
```

ESM 実行時の解決はブラウザ仕様準拠で、拡張子補完が一切効かない。`.ts` を書いていても import 文には**出力後の `.js`** を書く。

```diff
-import { slug } from "./util";
+import { slug } from "./util.js";
```

tsconfig で `"moduleResolution": "NodeNext"` を入れると、この漏れを `tsc --noEmit` の時点で `error TS2835` として検出でき、実行前に潰せる。

## verbatimModuleSyntax で import type 漏れを検出する

型だけの import が値 import に混ざると、ランタイムに不要なモジュールが残り CJS/ESM 境界で落ちる。`verbatimModuleSyntax: true` は型 import を構文レベルで強制する。

```json
{ "compilerOptions": { "verbatimModuleSyntax": true, "module": "NodeNext" } }
```

```diff
-import { User } from "./model.js";        // error TS1484
+import type { User } from "./model.js";   // 型は出力から完全に消える
```

これで「型のつもりが実 import されていた」事故が tsc 段階で 0 件に固定できる。

## import.meta.url で __dirname / __filename を復元

ESM には `__dirname` が無く、参照すると `ReferenceError: __dirname is not defined`。`node:url` と `node:path` で 2 行復元する。

```ts
import { fileURLToPath } from "node:url";
import { dirname } from "node:path";

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
// data ファイルの相対解決もこれで CJS と同じ感覚に戻る
import { join } from "node:path";
const cfg = join(__dirname, "config.json");
```

Node 20.11 以降なら `import.meta.dirname` が直接使え、上記 2 行は `const __dirname = import.meta.dirname;` に短縮できる。20.11 未満を CI に含むなら `fileURLToPath` 版を残すのが移植性の高い選択になる。
