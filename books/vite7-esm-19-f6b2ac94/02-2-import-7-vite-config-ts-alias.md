---
title: "第2章 import落ち7パターンとvite.config.tsのalias設定"
free: false
---

## `Failed to resolve import "./utils"` は拡張子省略が原因（修正1行）

Vite 7 の esbuild は ESM 準拠でモジュール解決するため、`import { fmt } from './utils'` のように拡張子を省くと `[plugin:vite:import-analysis] Failed to resolve import` を吐く。CommonJS 時代の癖で書いた相対 import がここで全滅する。

```ts
// NG: 解決失敗（utils.ts があっても落ちる場合がある）
import { fmt } from './utils'

// OK: 拡張子を明示。修正は末尾4文字の追記のみ
import { fmt } from './utils.ts'
```

検証は `npx vite build` 1回。`transforming...` で止まらず `✓ built` が出れば解決。7パターン中、この拡張子起因が体感5割を占める。

## `@/components` が解決できない時の resolve.alias 正規記法

`@` を `src` に貼る設定は3行で済むが、`path.resolve` の基準ディレクトリを間違えると `Failed to resolve import "@/components/Btn"` が消えない。`__dirname` ではなく `process.cwd()` を使うとモノレポで破綻する。

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import { fileURLToPath, URL } from 'node:url'

export default defineConfig({
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url)),
    },
  },
})
```

`fileURLToPath(new URL('./src', import.meta.url))` は ESM の `vite.config.ts` で唯一安定する書き方。`__dirname` は ESM 設定ファイルでは undefined になり、ここで2次エラーを誘発する。

## alias は通るのに型だけ赤線が出る：tsconfig.json の paths 併記漏れ

Vite 側 alias と TypeScript の型解決は別系統。`vite.config.ts` だけ直すと実行は通るが VSCode で `Cannot find module '@/components/Btn' or its corresponding type declarations.(2307)` が残る。原因は `tsconfig.json` の `paths` 未記載。

```jsonc
// tsconfig.json — baseUrl と paths を必ずペアで書く
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

検証は `npx tsc --noEmit`。エラー0行になれば赤線も消える。alias を足すたび `paths` を足す——この二重管理を忘れると「動くのに赤い」矛盾に毎回ハマる。

## `Directory import is not supported` を消す index 解決の落とし穴

`import App from './App'` で `App/index.tsx` を読ませる書き方は、Vite 7 + `"type": "module"` 下で `Directory import './App' is not supported resolving ES modules` を出すことがある。ディレクトリ index の暗黙解決は ESM では非標準扱い。

```ts
// NG: ディレクトリ暗黙 index
import App from './App'

// OK: index まで明示するか extensions を拡張
import App from './App/index.tsx'
```

設定で吸収するなら `resolve.extensions` に `.tsx` を含めるが、デフォルト配列を上書きして `.mjs` を落とすと別 import が壊れる。追加時は既存値を残す：

```ts
resolve: { extensions: ['.mjs', '.js', '.ts', '.jsx', '.tsx', '.json'] }
```

## 7パターン症状別逆引き表と15分復旧フロー

エラーメッセージから該当行へ飛ぶための対応表。相対パス地獄はこの順で潰すと平均15分で収束する。

```bash
# 復旧フロー（コピペで上から実行）
npx vite build        # ① import-analysis エラーの行番号を特定
npx tsc --noEmit      # ② 型側 paths 漏れ(2307)を特定
grep -rn "from '\.\./\.\./" src   # ③ 相対パス3階層以上を alias 化候補として列挙
```

| エラー文の一部 | 該当ファイル | 修正 |
|---|---|---|
| `Failed to resolve import` | import 文 | 拡張子 `.ts/.tsx` 追記 |
| `@/... 解決不可` | vite.config.ts | `resolve.alias` 追加 |
| `(2307)` 型だけ赤線 | tsconfig.json | `paths` 併記 |
| `Directory import not supported` | import 文 | `/index.tsx` 明示 |
| `Unknown file extension` | resolve.extensions | 既存配列に追記 |

③の `grep` で3階層以上の `../` をゼロにし、すべて `@/` へ寄せれば、import 起因の再発は実測でほぼ止まる。
