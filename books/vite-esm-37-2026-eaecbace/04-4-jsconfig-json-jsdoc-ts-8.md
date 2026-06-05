---
title: "第4章 jsconfig.json + JSDocで型を効かせる: TS未導入プロジェクトの設定エラー8件"
free: false
---

## エラー文字列で引く: jsconfig特有8件の対応表

VS Codeのコンソールや問題パネルに出る文字列をキーに、まず該当行へ飛ぶ。Vite本体は`.js`をそのまま通すため、ここでの8件はすべて**エディタ（tsserver）側の警告**である点を最初に切り分ける。

```text
1 "Cannot use JSX unless the '--jsx' flag is provided"     → compilerOptions.jsx 未設定
2 "Could not find a declaration file for module 'xxx'"      → checkJs + 型なし依存
3 "Property 'x' does not exist on type ..."                 → JSDoc型不足
4 "An import path cannot end with a '.ts' extension"        → allowImportingTsExtensions
5 "Option 'allowJs' ... requires ..."                       → allowJs/checkJs の組合せ
6 "module is not defined in ES module scope"               → module/moduleResolution
7 "Cannot find module './x' or its type declarations"      → paths/baseUrl
8 "Implicitly has an 'any' type"                            → checkJs有効時のno-implicit挙動
```

## allowJs/checkJs エラー(#1,#5): Vite ESM最小jsconfig

JS専用プロジェクトで型恩恵だけ得る最小構成。Viteのビルドには一切影響せず、`tsserver`が`.js`を解析するためだけに置く。

```jsonc
// jsconfig.json (プロジェクト直下)
{
  "compilerOptions": {
    "checkJs": true,        // .js を型チェック対象に
    "module": "ESNext",
    "moduleResolution": "Bundler", // Vite 4+/5 向け
    "target": "ES2022",
    "jsx": "preserve",      // #1 を解消
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  },
  "include": ["src/**/*.js", "src/**/*.jsx"],
  "exclude": ["node_modules", "dist"]
}
```

`jsconfig.json`では`allowJs`は常に`true`扱いのため明示不要。`tsconfig.json`へ移す場合のみ`allowJs: true`を書く（#5の正体はこの差）。

## 型なし依存エラー(#2): JSDoc + d.ts 24行で黙らせる

`Could not find a declaration file`はAI生成コードが無名importを吐いたときに頻発する。型を捏造せず`unknown`で受けるのが安全。

```js
// src/types/shims.d.ts
declare module "untyped-pkg";   // 最短: any 扱いで沈黙

// 個別に型を当てる場合
declare module "tiny-emitter" {
  export default class Emitter {
    on(e: string, fn: (...a: unknown[]) => void): void;
    emit(e: string, ...a: unknown[]): void;
  }
}
```

```js
// AI生成関数をJSDocで軽くガード
/** @param {{id:number, name:string}} u */
export function label(u) {
  return `${u.id}:${u.name}`; // u.nme で #3 が即赤線
}
```

## エディタ赤線/Vite通過の不一致(#6): moduleResolution整合

「VS Codeだけ赤、`vite dev`は成功」は`moduleResolution`がVite実体と食い違うパターン。Vite 5は内部でesbuild bundler解決を使うため`Bundler`に揃える。`node`のままだと`@/*`が解決できず#7が出る。

```bash
# 不一致の確認: tsserverと同じ条件でCLI型チェック
npx tsc --noEmit -p jsconfig.json
# → ここで0件ならエディタの赤線はキャッシュ。下記で再起動
# VS Code: Ctrl+Shift+P → "TypeScript: Restart TS Server"
```

```jsonc
// vite.config.js 側のエイリアスとjsconfig.paths を一致させる
import { fileURLToPath, URL } from "node:url";
export default {
  resolve: { alias: { "@": fileURLToPath(new URL("./src", import.meta.url)) } }
};
```

## tsconfig対応表: どこからがビルドに影響するか

`jsconfig.json`は「`compilerOptions.checkJs`を暗黙`true`にした`tsconfig.json`」に等しい。Viteは両者の`paths`を**読まない**（解決は`vite.config.js`の`alias`のみ）。境界を一覧化する。

| フィールド | jsconfigでの役割 | ビルド影響 |
|---|---|---|
| `checkJs` | エディタ型チェック | なし |
| `jsx` | JSX赤線抑止 | なし(esbuildが変換) |
| `paths`/`baseUrl` | 補完・定義ジャンプ | なし(aliasが担当) |
| `target` | 構文解析の許容 | なし |
| `strict` | #8の厳格度 | なし |

```bash
# TS移行時の差分: リネームしてallowJsを足すだけ
mv jsconfig.json tsconfig.json
npx json -I -f tsconfig.json -e 'this.compilerOptions.allowJs=true'
# include を *.ts へ拡張し、ファイルを段階的に .ts へ
```

これで8件すべてが**ビルド非影響＝エディタ設定で完結**と確定し、`vite dev`が通る限り赤線はTS Server再起動か`moduleResolution`調整で90秒以内に消せる。
