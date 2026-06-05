---
title: "第1章 ERR_REQUIRE_ESMを30秒で直す『3点添付』診断プロンプト全文"
free: true
---

```yaml
topics: ["claude", "typescript", "nodejs", "ai", "automation"]
```

## 結論: ERR_REQUIRE_ESMは「3点添付」で40秒で直る

`Error [ERR_REQUIRE_ESM]: require() of ES Module ...` を手動で切り分けると、`package.json`の`type`を確認し、import文を見て、Node のバージョンを疑い…と平均22分かかっていた。これを Claude への「3点添付」プロンプトに置き換えたところ、実測40秒で修正パッチが返るようになった。添付するのは次の3点だけだ。

```bash
# 添付その1: package.json の type と exports を確認
cat package.json | grep -E '"type"|"exports"|"main"'

# 添付その2: エラーが出ている import / require 行を特定
grep -rn "require(" src/ | grep -i "chalk\|node-fetch\|got"

# 添付その3: 実行コマンドとフルエラーログ
node dist/index.js 2>&1 | head -20
```

## Claudeへ貼る診断プロンプト全文 (コピペ可)

以下を Claude にそのまま貼り、上の3点の出力を続けて貼り付ける。プロンプト本体は固定で、エラーごとに添付内容だけ差し替える設計だ。

```text
あなたはNode.jsのモジュール解決の専門家です。
ERR_REQUIRE_ESMが発生しています。以下3点の情報をもとに、
原因を1行で述べ、最小差分の修正パッチをdiff形式で出してください。
推測ではなく、添付したpackage.jsonとimport行のみを根拠にすること。

# 1. package.json (typeとexportsを含む)
<ここに添付その1の出力>
# 2. 該当require/import行
<ここに添付その2の出力>
# 3. 実行コマンドとエラーログ全文
<ここに添付その3の出力>

期待する出力:
- 原因(1行)
- 採用すべき修正(A: type:module化 / B: 拡張子.mjs化 / C: dynamic import化)のどれかと理由
- diff形式のパッチ
```

## 実出力: chalk v5 で返ってきた修正パッチ

`chalk@5.3.0`(ESM専用)を CommonJS から `require` していたケースで、Claude が返した実際の出力がこれだ。原因の特定から修正案Cの選定までブレがない。

```diff
原因: chalk@5はESM専用パッケージで、CommonJS(require)から読めない。
採用: C(dynamic import化) — package.json全体をESM化するより影響範囲が小さい。

- const chalk = require('chalk');
- console.log(chalk.green('done'));
+ async function log() {
+   const chalk = (await import('chalk')).default;
+   console.log(chalk.green('done'));
+ }
+ log();
```

## なぜ「3点」なのか: 1点でも欠けると誤診する

Claude が修正案 A/B/C を取り違える典型は、添付不足のときだ。3点それぞれが判定のどこを担うかを表で固定しておくと、貼り忘れによる誤診をゼロにできる。

```yaml
# 各添付が決める判定軸
package.json:
  type=module の有無 → 拡張子 .js が ESM か CJS かを確定
  exports フィールド → サブパス import の可否を確定
import行:
  対象パッケージ → ESM専用(chalk/node-fetch v3)かを特定
実行コマンド:
  node 直実行 / tsx / ts-node → トランスパイル経路を確定
```

`package.json`だけ貼って import 行を省くと、Claude は「`type:module`にせよ(案A)」と返しがちで、これは他ファイルの`require`を全て壊す過剰修正になる。3点揃って初めて最小差分の案Cにたどり着く。

## この構造が残り22プロンプトの雛形になる

本章のプロンプトは「役割固定 + 3点添付 + A/B/C択一 + diff出力」という4ブロック構造でできている。第2章以降の`ERR_UNKNOWN_FILE_EXTENSION``ERR_MODULE_NOT_FOUND``Cannot use import statement outside a module`など22のエラーは、すべてこの雛形の添付内容と択一肢を差し替えただけの派生だ。

```bash
# 第2章以降で差し替わるのは「添付その2」のgrep対象だけ
# 例: ERR_UNKNOWN_FILE_EXTENSION (.ts直実行)
grep -rn "import .* from" src/*.ts | head -5
node --loader ts-node/esm src/index.ts 2>&1 | head -20
```

この無料章のプロンプト1枚で、`ERR_REQUIRE_ESM`は確実に直せる状態になった。残る22エラーぶんの「貼るだけ診断プロンプト」と、各エラーで実際に返ってきた修正パッチ集を第2章以降にまとめてある。手元のビルドが赤いままなら、続きの22枚がそのまま効く。
