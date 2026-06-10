---
title: "第1章: ERR_REQUIRE_ESMをClaudeに貼って30秒で直す最初の1プロンプト"
free: true
---

## ERR_REQUIRE_ESMの正体は「typeかimport/requireの不一致」の2択でしかない

`ERR_REQUIRE_ESM`が出たら、原因は2つに絞られる。`package.json`の`"type": "module"`とファイルの読み込み方法が食い違っているか、CommonJSのファイルからESM専用パッケージを`require()`しているかだ。Node.js 18以降で最頻出のこのエラーは、tsconfigやバンドラの知識ゼロでも、エラー全文をClaudeに貼れば40秒で切り分けが終わる。

まずは典型的なエラー全文を確認しておく。

```bash
$ node index.js
node:internal/modules/cjs/loader:1289
  throw new ERR_REQUIRE_ESM(filename, true, parentPath);
        ^
Error [ERR_REQUIRE_ESM]: require() of ES Module /app/node_modules/chalk/source/index.js
from /app/index.js not supported.
Instead change the require of index.js in /app/index.js to a dynamic import()
    code: 'ERR_REQUIRE_ESM'
```

## 手探りの18分を40秒に縮めた「3点セット」テンプレ

著者が最初にこのエラーに当たったとき、StackOverflowとchalkのCHANGELOGを行き来して18分を溶かした。縮めた決め手は、Claudeに渡す情報を「エラー全文・`package.json`・該当行」の3点に固定したことだ。この3つが揃うとClaudeは推測せず断定できる。

コピペで使える最初の1プロンプトがこれだ。

```text
次のNode.jsのエラーを直してください。原因が「package.jsonのtypeフィールド」か
「require/importの不一致」かを最初に1文で断定し、その後に修正diffを出してください。

【エラー全文】
Error [ERR_REQUIRE_ESM]: require() of ES Module /app/node_modules/chalk/source/index.js ...

【package.json】
{ "name": "app", "version": "1.0.0", "main": "index.js" }

【該当行 index.js:1】
const chalk = require('chalk');
```

## Claudeの実出力は「断定1文 → diff」の順で返ってくる

上のプロンプトを貼ると、Claudeは余計な前置きを挟まず原因を1文で断定する。chalk v5系はESM専用なので、CommonJSの`require()`では読めない、という判定だ。

実際の返答冒頭がこれ。

```text
原因: require/importの不一致です。chalk@5はESM専用パッケージのため、
CommonJS形式のrequire()では読み込めません。package.jsonのtypeは無関係です。
```

続けてClaudeは2通りの修正方針（ESMへ移行 / chalk v4へダウングレード）を提示する。プロジェクトを壊さない方を選べる。

## Claudeの修正diffを`git apply`で当てれば30秒で解決する

Claudeが返すdiffは`git apply`可能な形に整っている。ESMへ寄せる方針なら、`package.json`に`"type": "module"`を足し、`require`を`import`へ書き換える2箇所のdiffが返る。

```diff
--- a/package.json
+++ b/package.json
@@ -1,5 +1,6 @@
 {
   "name": "app",
   "version": "1.0.0",
+  "type": "module",
   "main": "index.js"
 }

--- a/index.js
+++ b/index.js
@@ -1 +1 @@
-const chalk = require('chalk');
+import chalk from 'chalk';
```

```bash
$ node index.js   # 再実行
Hello world       # ERR_REQUIRE_ESM が消えた
```

## ERR_REQUIRE_ESMと同じ型で残り46種のNode.jsエラーを潰す逆引き辞書

`ERR_REQUIRE_ESM`で証明された型は「エラー全文・設定ファイル・該当行の3点 → 原因の断定 → 修正diff」だ。この本の第2章以降は、`ERR_MODULE_NOT_FOUND`や`Cannot use import statement outside a module`など46種を、すべて同じ型の逆引きプロンプトとして辞書化している。

検索用に、自分のエラーがどの章かを30秒で当てるシェルスニペットを置いておく。

```bash
# 出たエラーコードを抜き出して章番号を引く
node app.js 2>&1 | grep -oE 'ERR_[A-Z_]+' | head -1
# → ERR_MODULE_NOT_FOUND なら第3章、ERR_UNKNOWN_FILE_EXTENSION なら第7章へ
```

この章のプロンプトをコピーして手元の`ERR_REQUIRE_ESM`に当てれば、次に同じエラーで詰まる時間はゼロになる。残り46種を同じ40秒で潰したい人は、第2章から症状別の逆引き辞書に進んでほしい。
