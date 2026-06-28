---
title: "【2026年最新】Node.jsのCommonJS/ESM地獄をClaudeに一発診断させるプロンプト集と自作ツール実装"
emoji: "🛠"
type: "tech"
topics: ["javascript","nodejs","esm","claude"]
published: true
---
> 本記事は筆者がQiitaで公開した記事のミラーです（原文: https://qiita.com/1280itsuya/items/1543ea6e9adfec641cc5 ）。

この記事を読み終えると、`Cannot use import statement outside a module` や `ERR_REQUIRE_ESM` が出た瞬間に、**エラー文・package.json・拡張子・Nodeバージョンを自動収集してClaude APIに投げ、原因と修正diffを返す自作CLI**が手元で動かせるようになります。コピペして`node diagnose.mjs エラーログ`で使える実コード2本付きです。

## なぜCommonJS/ESMエラーはClaudeに丸投げした方が速いのか

**結論: このエラーは「文面」だけでは原因が一意に決まらず、`package.json`の`type`・ファイル拡張子・Nodeバージョン・呼び出し側/呼ばれ側の組み合わせという4変数を突き合わせないと診断できないから、文脈を機械的に集めてLLMに渡すのが最短です。**

例えば `Cannot use import statement outside a module` という同じ一文でも、真因は最低3パターンに分岐します。

1. `package.json`に`"type": "module"`が無く、`.js`がCommonJS扱いされている
2. `.js`を`import`で読もうとしたが、その依存パッケージ側がCJSのみ提供
3. ts-node / Jest など別のローダーがESM変換を噛んでおらず、トランスパイル前の`import`が素通りした

人間は「またこれか」とエスパーしますが、当たれば速い・外せば30分溶ける賭けです。だったら4変数を確実に集めてClaudeに渡し、賭けの部分をAIに肩代わりさせます。Stack Overflowを5タブ開く前に、`type`フィールドと拡張子という決定的な証拠を最初に揃えるのがこの記事のキモです。

## Node.jsの環境コンテキストを機械収集する診断コレクタ

LLMに投げる前に、**エラー文だけでなく判断材料を全部集める**スクリプトを作ります。これがあると診断精度が露骨に上がります。`context.mjs`として保存してください。

```js
// context.mjs — CJS/ESM診断に必要な4変数を収集する
import { readFileSync, existsSync } from "node:fs";
import { join, extname } from "node:path";

export function collectContext(targetFile) {
  const ctx = {
    nodeVersion: process.version,
    targetExt: targetFile ? extname(targetFile) : null,
    packageType: "(未指定=CommonJS扱い)",
    hasTsConfig: existsSync("tsconfig.json"),
    exportsField: null,
  };

  const pkgPath = join(process.cwd(), "package.json");
  if (existsSync(pkgPath)) {
    const pkg = JSON.parse(readFileSync(pkgPath, "utf8"));
    ctx.packageType = pkg.type ?? "(未指定=CommonJS扱い)";
    // exportsの条件分岐(require/import)はデュアルパッケージ事故の温床なので拾う
    ctx.exportsField = pkg.exports ?? null;
    ctx.depCount = Object.keys(pkg.dependencies ?? {}).length;
  }
  return ctx;
}

// 単体実行: node context.mjs ./src/index.js
if (import.meta.url === `file://${process.argv[1]}`) {
  console.log(JSON.stringify(collectContext(process.argv[2]), null, 2));
}
```

ここで地味に効くのが`exports`フィールドの収集です。`"exports": { "require": "./cjs.js", "import": "./esm.js" }` のデュアルパッケージ構成は、`require`版と`import`版で**別インスタンスが2回ロードされて`instanceof`が壊れる**という、エラーすら出ない最悪の事故を起こします。これは文面に現れないので、構造として渡さないとClaudeも気づけません。

また`import.meta.url === \`file://${process.argv[1]}\`` は、ESMで`require.main === module`の代わりに「直接実行されたか」を判定する定番イディオムです。ESMには`__dirname`も`require.main`も無いため、ここで詰まる人が多い。Node 20.11 / 21.2以降なら`import.meta.dirname`が使えるので、`__dirname`が無いと言われたらまずバージョンを確認してください。

## Claude APIにエラーと文脈を渡して修正diffを受け取る

本体です。`@anthropic-ai/sdk`を使い、収集したコンテキストとエラー文をまとめてClaudeに投げます。`diagnose.mjs`として保存。

```js
// diagnose.mjs — node diagnose.mjs "<エラー文>" ./src/index.js
import Anthropic from "@anthropic-ai/sdk";
import { collectContext } from "./context.mjs";

const errorText = process.argv[2] ?? "";
const targetFile = process.argv[3] ?? null;
const ctx = collectContext(targetFile);

const client = new Anthropic(); // ANTHROPIC_API_KEY を環境変数から読む

const prompt = `あなたはNode.jsのモジュール解決の専門家です。
以下のエラーと環境から、CommonJS/ESMのどの不整合かを断定し、
最小の修正diffを1つだけ提示してください。憶測の選択肢列挙は禁止。

## エラー文
${errorText}

## 収集した環境(これが判断の根拠)
- Nodeバージョン: ${ctx.nodeVersion}
- 対象ファイル拡張子: ${ctx.targetExt}
- package.jsonのtype: ${ctx.packageType}
- exportsフィールド: ${JSON.stringify(ctx.exportsField)}
- tsconfig.json: ${ctx.hasTsConfig ? "あり" : "なし"}

## 出力形式(厳守)
1. 真因(1文): 
2. 修正方法(package.json編集 / 拡張子変更 / import文修正 のどれか1つ): 
3. 適用するdiff(\`\`\`diff ブロック): 
4. やってはいけない代替案(なぜ悪化するか): `;

const msg = await client.messages.create({
  model: "claude-sonnet-4-6", // 速くて十分。難物はclaude-opus-4-8に上げる
  max_tokens: 1500,
  messages: [{ role: "user", content: prompt }],
});

console.log(msg.content[0].text);
```

プロンプトで**「憶測の選択肢列挙は禁止」「diffを1つだけ」**と縛っているのが意図的です。これを書かないと、Claudeは親切心で「方法A、方法B、方法Cがあります」と返してきて、結局自分で選ばされる=Stack Overflowと同じ状態に逆戻りします。環境変数を根拠として明示的に渡しているので、「`type`が未指定だから`.js`がCJS扱いされている」と**証拠ベースで断定**させられます。

モデルは`claude-sonnet-4-6`を既定にしています。CJS/ESM診断は推論より知識照合のタスクなので速いモデルで十分で、`exports`の条件分岐が絡む複雑な事故だけ`claude-opus-4-8`に上げる運用が費用対効果が良いです。

## ts-nodeとJestで踏んだ実際の落とし穴

このツールを使い回す中で、文面だけ見ると真逆の診断をしかける典型例を挙げます。

`ts-node`で`SyntaxError: Cannot use import statement outside a module`が出たケース。素直に読むと「`package.json`に`type: module`を足せ」と言いたくなりますが、**それをやるとTypeScript側のCJS出力と衝突して別エラーに化けます**。真因はts-nodeのESMローダー(`--loader ts-node/esm` または`tsconfig`の`module`設定)が噛んでいないことで、`package.json`ではなく`tsconfig.json`の`module`/`moduleResolution`が戦犯でした。`context.mjs`で`hasTsConfig: true`を渡しておくと、Claudeがこの分岐を踏んで「package.jsonを触るな、tsconfigを見ろ」と返してくれます。**tsconfigの存在をコンテキストに入れる**という一手間が、ここで効きます。

もう一つはNodeバージョン依存の罠。「`require()` of ES Module ... not supported」は長らく『ESMはrequireできない』が定説でしたが、Node.js 22.12.0以降では`require(esm)`がフラグ無しで解禁されています。古い記憶のまま「`import()`に書き換えろ」と提案するAIや記事もありますが、`ctx.nodeVersion`を渡しておけば、Claudeに「あなたのNodeは22.12以上なので、その依存は実はrequireで通る。書き換え不要」と現実に即した回答をさせられます。バージョンという数値を握らせるかどうかで、提案の正しさが変わります。

## GitHub ActionsのCIでESMエラーを事前に潰す検査

最後に、このエラーをローカルではなくCIで先回りして検出する実用ネタ。`package.json`の`type`と実ファイルの拡張子が矛盾していないかを、GitHub Actionsで機械チェックします。

```yaml
# .github/workflows/module-check.yml
name: module-consistency
on: [push, pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "22" }
      - name: type と拡張子の矛盾を検出
        run: |
          TYPE=$(node -p "require('./package.json').type || 'commonjs'")
          echo "package type: $TYPE"
          # type:module なのに require( を使っている .js を洗い出す
          if [ "$TYPE" = "module" ]; then
            if grep -rnE "require\\(" --include="*.js" src/; then
              echo "::error::type:moduleなのに.jsでrequire()を使用。.cjsへ改名するかimportに変更を"
              exit 1
            fi
          fi
```

`node -p`で`package.json`の`type`を取り出し、`module`なら`.js`内の`require(`を`grep`で検出して落とします。`type:module`環境で`require`を使うと`ReferenceError: require is not defined in ES module scope`になりますが、これは実行時まで気づきにくい。CIの`grep`一発で、レビュー前に機械的に弾けます。診断ツールが「治療」なら、こちらは「予防接種」です。

## このツール集が結局やっていること

CJS/ESMエラーの厄介さは、エラー文が原因を一意に指さないことに尽きます。だから`context.mjs`で`type`・拡張子・Nodeバージョン・tsconfig・exportsという**証拠**を機械的に揃え、`diagnose.mjs`でClaudeに「憶測禁止・diff1つ」と縛って断定させ、GitHub Actionsで再発を予防する——この3点セットで、エスパーゲームを証拠ベースの診断に置き換えます。まず`context.mjs`と`diagnose.mjs`をコピペして、次にエラーを踏んだとき`node diagnose.mjs "<エラー>" <ファイル>`を叩いてみてください。賭けに使っていた30分が返ってきます。

<!-- netgakari-dev-cta -->
---
### 🛠 関連リンク（筆者の制作物）

本記事のような **Claude / GitHub Actions を使った開発の自動化**を、すぐ自分の環境で動かしたい方へ:

- AI開発の自動化キット・プロンプト集（コピペで動く設定とCLAUDE.md実例集）→ https://itsuya.gumroad.com/l/agentrules260619
- 開発で詰まったエラーを即解決・無料ツール集 **DevToolBox** → https://1280itsuya.github.io/devtools/

<small>※自社商品・サイトへのリンク（プロモーションを含みます）。</small>
