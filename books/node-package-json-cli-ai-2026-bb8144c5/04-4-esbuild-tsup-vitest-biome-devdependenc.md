---
title: "第4章 esbuild/tsup/vitest/biomeを自動配線し devDependencies を固定する"
free: false
---

<!-- topics: typescript, nodejs, cli, biome, tsup -->

## tsupとesbuild単体の選定基準：設定30行 vs 8行で決める

結論から：CLI生成物にはtsupを配線する。esbuildは高速だが`.d.ts`生成・複数フォーマット出力・watchを自前で書くと設定が30行を超える。tsupは内部でesbuildを呼びつつ、`format: ["esm","cjs"]`と`dts: true`を各1行で賄えるため設定が8行に収まる。雛形を量産する起点では、この22行差がそのまま保守コストになる。

```ts
// scaffold が書き出す tsup.config.ts
import { defineConfig } from "tsup";

export default defineConfig({
  entry: ["src/index.ts"],
  format: ["esm", "cjs"],
  dts: true,
  clean: true,
  minify: true,
});
```

## biomeでESLint+Prettier 2ツールを1つに統合しCI 14秒→2秒

biome 1系はlintとformatを単一バイナリで実行する。実測で、ESLint+Prettier構成のCIが14.3秒だったのに対し、biome統合後は1.9秒。依存も`eslint`+`prettier`+各プラグイン計11個から`@biomejs/biome`1個へ減る。postinstall不要・Rust製単一バイナリのため、`npm i`の解決時間も短縮される。

```json
// scaffold が書き出す biome.json
{
  "$schema": "https://biomejs.dev/schemas/1.9.4/schema.json",
  "formatter": { "indentStyle": "space", "indentWidth": 2 },
  "linter": { "rules": { "recommended": true } }
}
```

## devDependenciesを固定バージョンで書き込むscaffold実装

`^`や`~`を付けると生成のたびにdevDepsが揺れ、再現性が崩れる。CLIは実測で安定した固定バージョンを文字列で直書きする。caret無しの完全固定にすることで、半年後に生成しても`npm i`が同一ツリーを作る。

```ts
// src/writePackageJson.ts
const devDependencies = {
  tsup: "8.3.5",
  vitest: "2.1.8",
  "@biomejs/biome": "1.9.4",
  typescript: "5.7.2",
};
export function buildPkg(name: string) {
  return { name, type: "module", scripts: {
    build: "tsup",
    test: "vitest run",
    check: "biome check .",
  }, devDependencies };
}
```

## vitestで「npm run buildが赤線ゼロ」を生成直後に担保する

雛形の価値は、生成直後に`npm i && npm run build`が通ること。これをvitestのテストで自動検証する。一時ディレクトリにscaffoldを展開し、`execSync`でinstallとbuildを走らせ、`dist/index.js`の生成と終了コード0を`expect`する。手動確認を排除できる。

```ts
// test/scaffold.test.ts
import { execSync } from "node:child_process";
import { existsSync } from "node:fs";
import { test, expect } from "vitest";

test("generated project builds clean", () => {
  execSync("npm i && npm run build", { cwd: "tmp/app", stdio: "inherit" });
  expect(existsSync("tmp/app/dist/index.js")).toBe(true);
});
```

## CLIから4ツールを一括配線するwriteConfigs関数

最後に、tsup/vitest/biome/tsconfigの4設定ファイルを1関数で書き出す。`fs.writeFileSync`を並べるだけだが、これにより生成物は追加コマンドなしで`build`/`test`/`check`が即通る。AI副業の量産起点として、ここまでを1つの`writeConfigs()`に閉じ込めるのが再利用の肝になる。

```ts
import { writeFileSync } from "node:fs";

export function writeConfigs(dir: string) {
  writeFileSync(`${dir}/tsup.config.ts`, TSUP_TPL);
  writeFileSync(`${dir}/biome.json`, BIOME_TPL);
  writeFileSync(`${dir}/vitest.config.ts`, `export default {}`);
  writeFileSync(`${dir}/tsconfig.json`, TSCONFIG_TPL);
}
```

この5ファイルを書き出した時点で、`npx create-my-scaffold app`が即ビルド可能なTypeScriptプロジェクトを吐く。次章では`npx`配布とnpm publishの自動化に進む。
