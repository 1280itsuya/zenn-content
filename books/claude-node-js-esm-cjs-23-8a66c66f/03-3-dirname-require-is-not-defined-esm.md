---
title: "第3章 __dirname/require is not definedをESMで蘇らせる修正パッチ生成プロンプト"
free: false
---

<!-- topics: claude, typescript, nodejs, ai, automation -->

ESM化した瞬間に `ReferenceError: __dirname is not defined` と `require is not defined` が同時に噴き出すのは、Node 20の標準ローダーがCommonJSのグローバルを一切注入しないからだ。この章は「なぜ消えるか」を理解させない。`import.meta.url` と `createRequire` を使う**修正パッチをClaudeに書かせる3プロンプト**を、添付ファイルと期待出力ごと配る。

## __dirnameを蘇らせるパッチ生成プロンプト(Node20前提を強制)

`__dirname` を `import("ESMで書いて")` 任せにすると、Claudeは古い `path.dirname(require.main.filename)` を返すことがある。`fileURLToPath` を名指しで縛る。

```text
このファイルはESM(.mjs / "type":"module")です。Node 20前提で
__dirname と __filename を復活させてください。制約:
- node:url の fileURLToPath を使用
- import.meta.url を起点にする
- TypeScript型注釈つき(string)
- 出力はパッチ部分のdiffのみ、解説不要
[添付] 該当ファイル全文
```

期待出力はこのdiffに固定される。

```diff
+import { fileURLToPath } from "node:url";
+import { dirname } from "node:path";
+
+const __filename: string = fileURLToPath(import.meta.url);
+const __dirname: string = dirname(__filename);
```

## requireをcreateRequireで再定義させる添付セット

`require("./config.json")` のようなCJS資産を残したいときは `module.createRequire` を使う。プロンプトに「動的import化しない」と明記しないとClaudeが勝手に `import()` へ書き換える。

```text
ESM内で同期require("./config.json")を維持したい。
node:module の createRequire を import.meta.url で初期化し、
既存のrequire呼び出しは書き換えずそのまま動かすパッチを出力。
[添付] エラーログ(require is not defined)+該当行
```

返るパッチ:

```ts
import { createRequire } from "node:module";
const require = createRequire(import.meta.url);
const config = require("./config.json"); // 既存行は無改変
```

## トップレベルawaitで詰まる動的requireの両対応プロンプト

`require` を `await import()` へ書き換えると、関数内 `require` がトップレベル `await` に化けて `await is only valid in async functions` で落ちる失敗が頻発する。1プロンプトで両対応させる。

```text
下記のrequire("./db.js")をimport()へ移行。ただし:
- 呼び出しが関数内なら await import() + 関数をasync化
- トップレベルなら createRequire を使い await を出さない
- どちらを選んだか1行コメントで明記
[添付] requireを含む関数全体
```

トップレベル判定時はawaitを避ける分岐が返る。

```ts
// top-level: createRequireで同期解決(TLA回避)
import { createRequire } from "node:module";
const require = createRequire(import.meta.url);
const { query } = require("./db.js");
```

## 生成パッチを貼って動いた実ファイルのdiff

実際に貼り付けて `node --version` v20.11 で通った `scripts/build.mts` のdiffがこれだ。3プロンプトの出力をそのまま連結しただけで、手書き修正はゼロだった。

```diff
-const out = path.join(__dirname, "dist");
-const tpl = require("./template.json");
+import { fileURLToPath } from "node:url";
+import { dirname, join } from "node:path";
+import { createRequire } from "node:module";
+const __dirname = dirname(fileURLToPath(import.meta.url));
+const require = createRequire(import.meta.url);
+const out = join(__dirname, "dist");
+const tpl = require("./template.json");
```

この3プロンプトをスニペット登録しておけば、次章の `ERR_REQUIRE_ESM` でも添付ファイルを差し替えるだけで同じ速度で潰せる。
