---
title: "第2章 ESM/CJS混在で require が爆発: package.json type と拡張子の正解"
free: false
---

## whereHas... いや、結論を3行で: type と拡張子の組み合わせは4通りしかない

`Cannot use import statement outside a module` も `require is not defined` も、原因は「package.json の `type` フィールド」と「ファイル拡張子」の組み合わせミス1点に収束する。下の対応表で自分のケースを2秒で特定し、`type` を消すか拡張子を `.mjs/.cjs` に変えれば直る。

```text
| type 値      | 拡張子 | import文 | require文 | 判定          |
|--------------|--------|----------|-----------|---------------|
| "module"     | .js    | OK       | 爆発(※1)  | requireを削除 |
| "commonjs"/無 | .js    | 爆発(※2) | OK        | typeを"module"|
| (任意)        | .mjs   | OK       | 爆発      | 常にESM       |
| (任意)        | .cjs   | 爆発     | OK        | 常にCJS       |
```

`.mjs` と `.cjs` は `type` を無視して挙動が固定される。混乱したら拡張子で殴るのが最短で、これがNode側不動バグの9割を占める。

## ※1 require is not defined: Claude SDK(ESM)とPlaywright同居で出た4エラー

`@anthropic-ai/sdk`(ESM配布) と Playwright製スクリプト(CJSで書かれていた)を同じ`scripts/post.js`に置いた瞬間、4種のエラーが出た。最小修正の文字数も記録しておく。

```diff
# エラーA: require is not defined in ES module scope  → 修正3文字
- const { chromium } = require('playwright');
+ import { chromium } from 'playwright';

# エラーB: Cannot use import statement outside a module → package.json 18文字追加
- {}
+ { "type": "module" }
```

エラーAは`require`→`import`で実質3文字、エラーBは`type`行の追加18文字で起動した。

## __dirname が消えた: ESM化で壊れる定番2行をコピペで戻す

`type: "module"` にすると`__dirname`と`require`が消える。Playwrightのスクショ保存パスを組む箇所が `__dirname is not defined` で死ぬので、ESM用の置換2行を貼る。

```js
import { fileURLToPath } from 'node:url';
import { dirname } from 'node:path';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
// これで existing の path.join(__dirname, 'shot.png') がそのまま動く
```

## tsconfig は module と moduleResolution を必ずペアで指定する

TypeScriptで書く場合、`module`単体を直しても`Cannot find module './x'`が残る。Node16以降は`module`と`moduleResolution`をセットで`NodeNext`にするのが正解で、`tsc`の警告が0件になる。

```jsonc
// tsconfig.json — Node実行用の最小構成
{
  "compilerOptions": {
    "module": "NodeNext",          // moduleだけ直しても不足
    "moduleResolution": "NodeNext",// このペアが必須
    "target": "ES2022",
    "esModuleInterop": true        // CJSパッケージのdefault import救済
  }
}
```

`--experimental-vm-modules`等のフラグで誤魔化すと、CIとローカルで挙動が割れて再発する。フラグではなく`tsconfig`で固定する。

## CJS依存を1つだけ残す: 動的import()と条件分岐ローダ

`type: "module"`へ移行後も、CJSしか出していない古いライブラリ(例: 一部の`canvas`系)が1つ残ることがある。トップレベル`import`では読めないので、動的`import()`に退避し、失敗時だけ`createRequire`でCJSロードする分岐を入れる。

```js
import { createRequire } from 'node:module';

async function loadLegacy(name) {
  try {
    return (await import(name)).default;   // まずESMとして試す
  } catch {
    const require = createRequire(import.meta.url);
    return require(name);                  // CJSのみのパッケージを救済
  }
}
const legacy = await loadLegacy('some-cjs-only-pkg');
```

この分岐ローダなら、ESM主体のまま不動のCJS依存を1つだけ生かせる。ここまでがNode側の切り分け。次章ではブラウザ側で`document is not defined`が出るパターンへ進む。
