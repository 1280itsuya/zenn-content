---
title: "第3章: import/require混在エラー20種を症状別に逆引きする診断プロンプト辞書"
free: false
---

この章では「エラー文をコピペ→専用プロンプトを貼る→Claudeの回答と修正diffをそのまま適用」の3手順で、import/require混在エラー20種を平均30秒で潰す逆引き辞書を提供する。各項目に検証済みのClaude (claude-opus-4-8) 出力と、`type: module`追加が別ファイルを壊す典型ハマりを防ぐ追い質問テンプレを付ける。

## `Cannot use import statement outside a module`を波及範囲込みで診断するプロンプト

結論から書く。このエラーは「CJSとして実行されたファイルでESM構文を使った」状態で、`type: module`を足すだけだと同一パッケージの他ファイルが`require is not defined`で連鎖崩壊する。だから単独修正ではなく波及範囲を列挙させる。

貼るプロンプトはこれ1つ。

```text
このエラーを修正したい。package.json全体とエラーが出たファイル名を渡す。
1. 原因を1行で
2. 修正方針を「最小変更」と「全面ESM移行」の2案で
3. 各案で require / module.exports / __dirname を使う他ファイルへの波及を全列挙
4. 推奨案の修正diffをunified形式で
{ここにpackage.jsonとエラー全文を貼る}
```

想定されるClaude回答の核は次のdiffになる。`type: module`を足すなら、同梱スクリプトの拡張子も同時に変える。

```diff
 {
   "name": "my-tool",
+  "type": "module",
   "scripts": {
-    "build": "node build.js"
+    "build": "node build.mjs"
   }
 }
```

## `__dirname is not defined`をESM等価コードへ変換する1行プロンプト

`__dirname`と`__filename`はCJSグローバルで、ESMには存在しない。Claudeに「ESM等価へ」と頼むと`import.meta.url`を使う定型へ即変換する。

```text
ESMで __dirname is not defined。下のCJSコードをESM等価へ。fileURLToPathを使い、
Node 20.11+なら import.meta.dirname も併記して。
{該当行を貼る}
```

修正diffはこの形に固定される。

```diff
+import { fileURLToPath } from 'node:url';
+import { dirname } from 'node:path';
+const __filename = fileURLToPath(import.meta.url);
+const __dirname = dirname(__filename);
 const config = path.join(__dirname, 'config.json');
```

## `Unexpected token 'export'`と`require() of ES Module`を方向別に切り分ける

この2つは矢印が逆だ。`Unexpected token 'export'`はESMファイルをCJSとして読んだ、`require() of ES Module`はCJSからESM専用パッケージを`require`した。Claudeに方向を明示させる。

```text
このエラーは「CJS→ESMを読んだ」「ESM→CJSを読んだ」どちらか判定して。
判定根拠（拡張子・package.jsonのtype・呼び出し側コード）も示し、
dynamic import への置換diffを出して。
```

`require` を `await import` に変える定番diff。

```diff
-const chalk = require('chalk');
+const { default: chalk } = await import('chalk');
```

## 誤診を捕まえる追い質問テンプレで再ハマりを防ぐ

Claudeが`"type": "module"`だけ提案して止まったら、そこで適用せず追い質問を打つ。これが30秒診断の検算手順だ。

```text
今の修正後、次を再チェックして差分を追加して：
1. jest / mocha 設定が require 前提で壊れないか
2. __dirname / module.exports を使う残りファイル
3. *.cjs に退避すべきスクリプト
4. CI（GitHub Actions）の node 実行コマンド
壊れる箇所があれば追加diffを、なければ「波及なし」と明記して。
```

この章の20プロンプトは付録Aの`prompts/ch3.json`に収録済みで、次のコマンドでエラー文キー一覧を引ける。

```bash
cat prompts/ch3.json | jq -r '.errors[].key'
# => cannot-use-import-statement
#    dirname-not-defined
#    unexpected-token-export ...全20件
```

次章ではこの逆引き辞書をVS Codeのスニペットへ落とし込み、エラー選択から3キーストロークで該当プロンプトを呼び出す。
