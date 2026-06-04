---
title: "第4章 tsx/Vitest/esbuildでESM混在を起動時に解決する設定"
free: false
---

## ts-nodeのESMローダーで `ERR_UNKNOWN_FILE_EXTENSION` が出る瞬間

`ts-node --esm` 系で最初に踏むのがこのエラー。Node.js 20.11 + ts-node 10.9 で `.ts` を直接起動すると停止する。

```bash
$ node --loader ts-node/esm src/index.ts
TypeError [ERR_UNKNOWN_FILE_EXTENSION]: Unknown file extension ".ts" for /app/src/index.ts
    at Object.getFileProtocolModuleFormat (node:internal/modules/esm/load:106:9)
```

原因は `--loader` が Node 20 で非推奨化され、ESMローダーの解決順が変わったこと。`package.json` に `"type": "module"` がある状態だと `.ts` の拡張子解決にローダーが間に合わない。ts-nodeでの延命より、起動ツールごと差し替えるほうが復旧が早い。

## tsx 4.7へ移行して `node --import tsx` で起動を安定させる

tsx は esbuild を内蔵し、`.ts` を実行時にトランスパイルする。起動失敗から成功までのコマンド履歴を残す。

```bash
$ npm i -D tsx@4.7.0
$ node --import tsx src/index.ts          # Node 20.6+ の --import で登録
✓ booted in 412ms

# package.json の scripts はこう変える
{
  "scripts": {
-   "dev": "node --loader ts-node/esm src/index.ts",
+   "dev": "tsx watch src/index.ts"
  }
}
```

`--loader` から `--import` への切り替えだけで `ERR_UNKNOWN_FILE_EXTENSION` は消える。tsx は CJS依存 (`require` 内包パッケージ) も `top-level await` も追加設定なしで通る。

## Vitestで `Cannot use import statement outside a module` をdeps.inlineで潰す

CJSのまま配布された古いパッケージを `import` するとテスト起動時に落ちる。実エラーはこれ。

```text
SyntaxError: Cannot use import statement outside a module
 ❯ node_modules/some-cjs-pkg/dist/index.js:1:1
```

`vitest.config.ts` で対象を inline 化し、Vitest側で変換させる。

```ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    server: {
      deps: {
        inline: ['some-cjs-pkg', /legacy-/],  // 正規表現で一括指定可
      },
    },
    deps: { optimizer: { ssr: { enabled: true } } },
  },
})
```

`inline` 指定後はトランスフォーム対象に入り、19 passed に戻る。`transformIgnorePatterns` 相当を Vitest では `server.deps.inline` が担う。

## esbuild/tsupでformat:esm,cjs両出力しトップレベルawaitの落とし穴を回避

ライブラリを両形式で配る場合、`tsup` (esbuild ラッパー) で2出力する。

```ts
// tsup.config.ts
import { defineConfig } from 'tsup'

export default defineConfig({
  entry: ['src/index.ts'],
  format: ['esm', 'cjs'],     // dist/index.mjs と dist/index.cjs を生成
  target: 'node20',
  dts: true,
})
```

ここで `top-level await` を含むと CJS 出力でだけ失敗する。

```text
✘ [ERROR] Top-level await is currently not supported with the "cjs" output format
```

CJS は構文上 `await` をトップレベルに置けない。回避は `format` を `['esm']` のみにするか、`await` を `async function main()` に包んで `main()` を呼ぶ形へ畳む。両出力を維持したいなら後者が正解。

## テストとビルドを同時に緑へ — 検証用ワンライナー

最後に起動・テスト・ビルドの3レーンが全部通る状態を1コマンドで確認する。

```bash
$ npx tsx src/index.ts && npx vitest run && npx tsup
✓ booted in 398ms
 Test Files  3 passed (3)
      Tests  19 passed (19)
ESM dist/index.mjs  4.21 KB
CJS dist/index.cjs  4.83 KB
⚡ Build success in 86ms
```

3レーンが緑なら ESM×CJS 混在はツール側で吸収できている。`tsx`(起動) / `Vitest deps.inline`(テスト) / `tsup format`(ビルド) の3点を押さえれば、`ERR_UNKNOWN_FILE_EXTENSION` と `Cannot use import statement` の両方を恒久的に閉じられる。手元の `package.json` の `scripts` をこの3コマンドへ揃えるところまでが本章の到達点。
