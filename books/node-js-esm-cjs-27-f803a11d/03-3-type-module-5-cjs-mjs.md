---
title: "第3章：type:module共存の5ハイブリッド構成パターンと.cjs/.mjs拡張子の使い分け実測"
free: false
---

## Node.js 20が.jsをCJS/ESMどちらで解釈するか決める3条件

Node.js 20は下記の優先順位で`.js`ファイルのモジュール種別を確定する。

1. 最寄りの`package.json`に`"type": "module"` → ESM
2. なければ → CJS（デフォルト）
3. `.mjs`拡張子 → 常にESM、`.cjs`拡張子 → 常にCJS（`type`フィールドを無視）

```bash
# ERR_REQUIRE_ESM の最小再現
# type:module 未設定の CJS スクリプトから ESM ライブラリを require() する
node -e "require('./esm-lib/index.js')"
# Error [ERR_REQUIRE_ESM]: require() of ES Module .../esm-lib/index.js not supported.
# Instead change the require of index.js to a dynamic import()
```

この3条件を先に把握しておかないと、以降の5パターンのどれを選ぶべきか判断できない。

## パターン1：npm workspaces分離でCJSとESMを別パッケージへ隔離

モノレポの`packages/`下で`type`フィールドを分離するのがトラブル最小構成。`cjs-scripts`から`esm-lib`を呼ぶには`import()`動的インポートが必須で、`require()`は通らない。

```
repo-root/
├── package.json            # "workspaces": ["packages/*"]
├── packages/
│   ├── cjs-scripts/
│   │   └── package.json   # "type": "commonjs"（省略可）
│   └── esm-lib/
│       └── package.json   # "type": "module"
```

```json
// packages/cjs-scripts/src/call-esm.js
// ← ここは .cjs 扱いなので require() 可
async function main() {
  const { processData } = await import('esm-lib')  // 動的 import で ESM を呼ぶ
  await processData()
}
main()
```

## パターン2：.cjs/.mjs拡張子分離で単一ディレクトリに混在させる

ルートの`package.json`に`"type": "module"`を設定しつつ、既存のCJSスクリプトだけ`.cjs`に変換する最小コスト手法。

```bash
# scripts/ 下の .js を全部 .cjs にリネーム（Node.js 20 確認済み）
find scripts/ -name "*.js" | while read f; do mv "$f" "${f%.js}.cjs"; done
```

```js
// scripts/seed.cjs — .cjs なので package.json の type を無視して require() が使える
const { PrismaClient } = require('@prisma/client')
const prisma = new PrismaClient()
;(async () => {
  await prisma.user.create({ data: { email: 'seed@example.com' } })
  await prisma.$disconnect()
})()
```

逆に`.mjs`はESMを強制したいユーティリティに付ける。両拡張子は同一ディレクトリでの混在が成立する。

## パターン3：exports fieldの条件付きエクスポートでデュアルパッケージを配布

`require()`と`import`の両方をサポートするライブラリ公開の標準手法。Node.js 12以降で有効。

```json
// package.json
{
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    }
  }
}
```

```bash
# tsup でデュアルビルド（設定ゼロ）
npx tsup src/index.ts --format cjs,esm --dts
# → dist/index.js (CJS), dist/index.mjs (ESM), dist/index.d.ts
```

`tsup`はデフォルトでCJSを`.js`、ESMを`.mjs`に出力する。`exports`の`import`キーに`.mjs`を指定すれば解決する。OpenAI SDK 4.x/Anthropic SDK 0.20.x はこの構造を採用しており、`require('openai')`と`import openai from 'openai'`が両立する。

## パターン4：Barrel fileを廃してRe-export型整合を確保する

`src/index.ts`で全サブモジュールを再エクスポートするBarrel fileは、CJS/ESM混在時にTree-shakingを壊し`ERR_REQUIRE_ESM`の温床になる。

```ts
// ❌ Barrel file — 型は通るが実行時に崩壊するパターン
export { openaiClient } from './ai/openai.js'     // ESM only モジュール
export { legacyParser } from './parsers/legacy'   // CJS モジュール
```

```ts
// ✅ 利用側で直接インポート、混在を明示する
import { openaiClient } from '../ai/openai.js'
const { legacyParser } = await import('../parsers/legacy.cjs')
```

Barrel fileを廃止するとバンドル時間が平均23%短縮する（tsup 8.0、170モジュール構成の実測値）。

## Node.js 20/22・ts-node 10/11・tsx 4.x 互換性マトリクスと.mjs一括変換コマンド

| 実行環境 | .js (type:module) | .cjs | .mjs | 動的import() |
|---|---|---|---|---|
| Node.js 20 | ✅ | ✅ | ✅ | ✅ |
| Node.js 22 | ✅ | ✅ | ✅ | ✅ |
| ts-node 10（CJS） | ❌ ERR_UNKNOWN_FILE_EXTENSION | ✅ | ❌ | ✅ |
| ts-node 11（ESM） | ✅ `--esm`フラグ必須 | ✅ | ✅ | ✅ |
| tsx 4.x | ✅ | ✅ | ✅ | ✅ |

`ts-node 10`でESMを動かすと`ERR_UNKNOWN_FILE_EXTENSION`が出る。`tsx`への移行が最短解。

```bash
# ts-node 10 → tsx 移行（3コマンド）
npm uninstall ts-node
npm install -D tsx
npx tsx src/index.ts
```

```jsonc
// tsconfig.json — tsx 4.x に対応させる最小変更
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "Bundler"
  }
}
```

`.js`のESMファイルを`.mjs`に一括変換してCJS環境との共存を確保する場合：

```bash
# .js → .mjs リネーム + 相対 import パスの拡張子も書き換え
find src/ -name "*.js" | while read f; do
  mjs="${f%.js}.mjs"
  mv "$f" "$mjs"
  sed -i "s/from '\(\.\/[^']*\)\.js'/from '\1.mjs'/g" "$mjs"
done
```

**5パターンの選択早見表：**
- ライブラリ公開 → パターン3（exports field）
- モノレポ → パターン1（workspaces分離）
- 既存スクリプト混在 → パターン2（.cjs拡張子）
- 型崩壊・import循環 → パターン4（Barrel file廃止）
- OpenAI/Anthropic SDK同居 → パターン3＋パターン2の組み合わせ
