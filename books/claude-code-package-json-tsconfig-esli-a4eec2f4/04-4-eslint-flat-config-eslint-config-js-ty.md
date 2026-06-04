---
title: "第4章 ESLint flat config(eslint.config.js)をtypescript-eslint対応で雛形化する"
free: false
---

## 結論: flat config移行は「テンプレ＋Claude検証」で事故が消える

`.eslintrc.json`からESLint v9のflat config（`eslint.config.js`）へ手動移行すると、9割が同じ4箇所で詰まる。本章のCLIは雛形を吐くだけでなく、Claude Code Agent SDKに`eslint .`の出力を読ませて自己修正させる。生成物が**警告0**で通るまでループする実装を作る。

```bash
$ node cli.js gen:eslint --tsconfig ./tsconfig.json
✓ eslint.config.js generated (typescript-eslint v8)
✓ self-check: npx eslint . → 0 problems
```

## typescript-eslint v8前提のparserOptions.projectとignoresを雛形化する

手動移行の最頻エラーは`parserが効かずno-unused-varsが全滅`だ。原因は`languageOptions.parser`の指定漏れと`project`の相対パス未解決。雛形側で`tsconfigRootDir`を絶対化して防ぐ。

```js
// templates/eslint.config.js
import tseslint from "typescript-eslint";
export default tseslint.config(
  { ignores: ["dist/**", "node_modules/**", "coverage/**"] },
  ...tseslint.configs.recommendedTypeChecked,
  {
    languageOptions: {
      parserOptions: {
        project: ["./tsconfig.json"],
        tsconfigRootDir: import.meta.dirname, // 相対パス事故を絶対化で封じる
      },
    },
  },
);
```

## globals未指定の「console未定義」をenv検出で自動付与する

`globals未指定でconsole is not defined`はflat config移行で必ず出る。`.eslintrc`の`env: { node: true }`が消えるためだ。CLIは`package.json`の`type`と依存を見てNode/Browserを判定し`globals`を注入する。

```js
import globals from "globals";
const pkg = JSON.parse(fs.readFileSync("package.json", "utf8"));
const isNode = !pkg.browser && !pkg.dependencies?.react;
const env = isNode ? globals.node : globals.browser; // console/processを解決
// → languageOptions.globals に展開
```

## eslint-config-prettierの競合ルールを末尾で自動除去する

PrettierとESLintのフォーマット系ルールが二重発火すると、保存のたびに差分が暴れる。`eslint-config-prettier`を**配列の最後**に置かないと無効化されない。CLIは挿入位置を強制する。

```js
import prettier from "eslint-config-prettier";
export default tseslint.config(
  ...baseRules,
  ...teamRules,
  prettier, // 必ず末尾。indent/quotes等の競合18ルールをoffにする
);
```

## 生成後にClaude Agent SDKでnpx eslintを回し自己修正させる

ここが本CLIの核心だ。生成して終わりにせず、`npx eslint .`の標準エラーをAgent SDKへ渡し、設定ミスをパッチさせて再実行する。最大3回ループで0警告に収束する。

```ts
import { query } from "@anthropic-ai/claude-code";
import { execSync } from "node:child_process";

for (let i = 0; i < 3; i++) {
  const out = execSync("npx eslint . 2>&1 || true").toString();
  if (out.includes("0 problems") || !out.trim()) break;
  for await (const _ of query({
    prompt: `eslint.config.js を修正。エラー全文:\n${out}\nparser/globals/ignoresのみ最小diffで直す`,
    options: { allowedTools: ["Edit"], cwd: process.cwd() },
  })) { /* SDKがファイルを直接編集 */ }
}
```

## チーム独自ルールを上書きせず差分マージする

再生成のたびにチームルールが消えると使われなくなる。`// @custom`マーカー間のブロックをASTではなく行範囲で保存し、雛形へ再注入する。

```js
const KEEP = /\/\/ @custom:start([\s\S]*?)\/\/ @custom:end/;
const prev = fs.existsSync("eslint.config.js")
  ? fs.readFileSync("eslint.config.js", "utf8").match(KEEP)?.[1] ?? ""
  : "";
const merged = template.replace("/* INJECT */", prev); // 独自ルールを温存
fs.writeFileSync("eslint.config.js", merged);
```

この6処理で、`.eslintrc`の暗黙挙動に依存していたプロジェクトもflat configへ無停止で移行できる。第5章ではこの自己修正ループを`tsconfig`生成にも横展開する。
