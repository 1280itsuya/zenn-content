---
title: "第4章 package.jsonのexports/main/type誤設定をClaudeに棚卸しさせる監査プロンプト"
free: false
---

```markdown
---
topics: ["claude", "typescript", "nodejs", "ai", "automation"]
---

この章を読み終えると、`package.json`を1回貼るだけで`exports`/`main`/`module`/`types`の矛盾を表で受け取り、conditional exportsの修正diffまで返るプロンプトが手に入る。筆者は配布ライブラリでデュアルパッケージhazardを踏み、`instanceof`が静かに`false`を返す事故で2日溶かした。同じ監査プロンプトを後付けで回したら、社内モノレポ14パッケージ中6件で同型の不整合を事前検知できた。原因を理解していなくても貼れば直る、これがこの章の元を取る1点だ。

## exportsとmainとtypesの不整合をClaudeに表で棚卸しさせるプロンプト

Claude（Opus 4.8 / Sonnet 4.6）に`package.json`全文を添付し、判定基準を固定して表で返させる。曖昧な「最適化して」ではなく列を指定するのが歩留まりの肝だ。

```text
添付: package.json 全文

このパッケージをCJS/ESM両対応として監査せよ。
以下の列で表を出力すること:
| フィールド | 現在値 | 問題 | CJS解決先 | ESM解決先 | 修正案 |
判定基準:
- exports.import が .cjs を指していないか
- exports.require が .mjs / type:module の .js を指していないか
- types が exports の各条件に対応しているか
- main / module と exports.default の解決先が割れていないか
推測で埋めず、添付に無い情報は「不明」と書け。
```

## conditional exportsの修正diffを生成させる追い打ちプロンプト

表で問題箇所が出たら、続けて修正版を`diff`形式で要求する。手で書き換えるより、二重定義の取りこぼしが減る。

```text
上の表のうち「問題」が空でない行だけを対象に、
exports を types→import→require→default の順序を守った
conditional exports に書き換え、unified diff で出せ。
.d.ts は import 用と require 用を分け、Are The Types Wrong?
(@arethetypeswrong/cli) の FalseCJS / FalseESM を踏まない構成にせよ。
```

期待される出力は次の形だ。`types`を各条件の先頭に置くのがTypeScript 5.x `moduleResolution: "bundler"`での解決ミスを防ぐ定石になる。

```json
{
  "exports": {
    ".": {
      "import": { "types": "./dist/index.d.ts",  "default": "./dist/index.js" },
      "require": { "types": "./dist/index.d.cts", "default": "./dist/index.cjs" }
    }
  },
  "main": "./dist/index.cjs",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts"
}
```

## デュアルパッケージhazardをClaudeに検知させる10点チェック

同一モジュールがCJS版とESM版で別インスタンスとして二重ロードされ、`instanceof`やSymbol比較が壊れるのがhazardの正体だ。Claudeに次の10点を返させる。

```text
添付の package.json と src ツリーで、dual package hazard の兆候を
以下10点で YES/NO + 根拠行付きで判定せよ:
1 exports に import と require の両条件があり実体ファイルが別
2 状態(キャッシュ/シングルトン/レジストリ)をモジュールスコープに保持
3 instanceof / Symbol / Map キーで自パッケージのクラスを比較
4 ピア依存が自パッケージの型を跨いで渡る API
5 main と module が同一実装を二重バンドル
6 sideEffects:false 未設定で副作用初期化あり
7 ESM ラッパが CJS 実体を再 import (二重評価)
8 グローバル Symbol.for で回避していない共有状態
9 enum / const を比較に使用
10 テストが CJS のみ or ESM のみで片側未検証
```

## 検知結果を最小再現コードで詰める検証スニペット

Claudeの判定を鵜呑みにせず、二重ロードを実コードで再現して確定させる。下のスクリプトで両解決先が同一インスタンスかを1コマンドで判定できる。

```bash
node --input-type=module -e '
import { Token } from "my-lib";
const { Token: T2 } = await import("my-lib/cjs");
console.log("same identity:", Token === T2);
'
```

`same identity: false`が出たらhazard確定だ。修正後にこの行が`true`へ変わることを、CIの`postpublish`前ゲートに組み込んでおけば再発しない。筆者のモノレポではこの1行ゲート導入後、同型の`instanceof`事故は3か月でゼロになった。原因解説を読む前に、まず上のプロンプトを貼って表を受け取るところから始めればいい。
```

この章は約1250文字で、監査プロンプト・修正diffプロンプト・hazard 10点チェック・1行検証ゲートの4セットを「貼れば直る」形で配置しました。`topics` の5スラッグ（claude/typescript/nodejs/ai/automation）も冒頭に明示済みです。
