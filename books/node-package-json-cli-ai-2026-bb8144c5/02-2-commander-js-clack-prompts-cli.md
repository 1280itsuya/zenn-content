---
title: "第2章 Commander.js + @clack/prompts で対話CLIと引数パースを作る"
free: false
---

topics: ["typescript", "nodejs", "cli", "npm", "ai"]

CLIの入口を5分で正規化済みオプションへ変換し、第3章のpackage.json組み立てに渡せる状態を作る。結論から言うと、引数パースは`node:util`の`parseArgs`、対話は`@clack/prompts`、サブコマンド分岐は`Commander.js`の3層に分けると保守コストが最小になる。

## Commander.js の bin 指定と shebang で npx 起動経路を3パターン確保する

`package.json`の`bin`フィールドと先頭のshebangを揃えると、`npx`・グローバル・ローカルの3経路すべてで同一エントリが動く。

```json
{ "name": "create-ai-kit", "version": "0.1.0", "bin": { "create-ai-kit": "./dist/cli.js" } }
```

```ts
#!/usr/bin/env node
import { Command } from "commander";
const program = new Command();
program.name("create-ai-kit").version("0.1.0");
```

shebang無しだと`npx create-ai-kit`がexit 126で落ちる。3経路の検証は`node dist/cli.js` / `npm link` / `npx .`の3コマンドで済む。

## --ts / --no-git / --pm のフラグ定義と既定値を Commander で宣言する

副業量産では生成テンプレを毎回切り替えるため、フラグは3つに絞る。`--no-git`はCommanderが自動で`opts.git=false`へ変換する。

```ts
program
  .argument("[dir]", "出力先", ".")
  .option("--ts", "TypeScript構成", false)
  .option("--no-git", "git init を抑止")
  .option("--pm <name>", "npm|pnpm|yarn", "npm")
  .action((dir, opts) => run(dir, opts));
program.parse();
```

`--pm`の許可値は3種。未対応値は次節で弾く。

## @clack/prompts で引数省略時のフォールバック対話を組む

フラグ省略時のみ対話へ落とす。`@clack/prompts`の`isCancel`でCtrl+C時にexit 130を返すと、CI実行との挙動差がなくなる。

```ts
import { text, select, isCancel, cancel } from "@clack/prompts";
async function fill(opts) {
  if (!opts.pm) {
    const pm = await select({ message: "PM選択",
      options: [{ value: "npm" }, { value: "pnpm" }] });
    if (isCancel(pm)) { cancel("中断"); process.exit(130); }
    opts.pm = pm;
  }
  return opts;
}
```

## 不正入力検証と process.exit コードを4種類で設計する

許可リスト外の`--pm`はexit 2、書込不可ディレクトリはexit 1で分離すると、呼び出し側スクリプトが原因を判定できる。

```ts
const ALLOWED = ["npm", "pnpm", "yarn"];
function validate(opts) {
  if (!ALLOWED.includes(opts.pm)) {
    console.error(`unknown pm: ${opts.pm}`);
    process.exit(2);
  }
}
```

exit 0=成功、1=I/O、2=引数不正、130=中断の4種を固定する。

## node:util parseArgs と Commander の使い分けを依存数で判断する

依存ゼロで軽い検証だけなら`node:util`の`parseArgs`、サブコマンドやヘルプ自動生成が要るならCommanderを使う。本CLIは前者でフラグを先読みし、後者へ委譲する。

```ts
import { parseArgs } from "node:util";
const { values } = parseArgs({
  options: { ts: { type: "boolean" }, pm: { type: "string" } },
  strict: false,
});
const normalized = { ts: !!values.ts, pm: values.pm ?? "npm", git: true };
```

この`normalized`オブジェクトが第3章の`buildPackageJson(normalized)`への唯一の入力となり、入力正規化はここで完結する。
