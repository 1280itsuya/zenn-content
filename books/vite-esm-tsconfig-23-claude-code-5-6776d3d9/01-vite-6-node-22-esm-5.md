---
title: "Vite 6+Node 22 ESM移行で必ず踏む地雷5種と即時リカバリコード【無料試し読み】"
free: true
---

Zenn章を執筆します。unique_angleの「Claude Codeプロンプトで診断→CIで再発防止」を章末の購買導線に組み込みます。

---

Vite 6 + Node 22 でESM移行した翌日のCIログがこうなっていたら、この章がそのまま答えになる。

```
SyntaxError: Cannot use import statement in a module
    at wrapSafe (node:internal/modules/cjs/loader:1472:18)
    at Module._compile (node:internal/modules/cjs/loader:1513:20)
```

5種の地雷を順に解体する。修正コードはすべてコピペで動く。

## 5エラーの発生源マップ — 修正前に30秒で全体像を把握

| # | エラー文字列（先頭40文字） | 真の原因 | 修正ファイル |
|---|---------------------------|----------|------------|
| ① | Cannot use import statement in a module | `"type":"module"` 未設定 | package.json |
| ② | ERR\_MODULE\_NOT\_FOUND | 相対importに `.js` 省略 | ソース全ファイル |
| ③ | Named export not found | バレルre-export + Vite tree-shake | index.ts |
| ④ | \_\_dirname is not defined | ESM非対応変数 | 該当ファイル |
| ⑤ | Top-level await is not allowed | tsconfig target < ES2022 | tsconfig.json |

④と⑤は同じ tsconfig 修正で同時解消できるため、実質4手で完了する。

## 地雷①: Cannot use import statement — package.json の1行追加で30秒解決

Node 22 は `"type": "module"` が無い場合、`.js` をCJSとして実行する。ESMの `import` 構文が入った瞬間にクラッシュする。

```bash
# エラー再現
node dist/index.js
# SyntaxError: Cannot use import statement in a module
```

```jsonc
// package.json — この1行を追加
{
  "name": "my-app",
  "type": "module",
  "scripts": { "dev": "vite", "build": "tsc && vite build" }
}
```

追加後、既存の CommonJS ファイル（`jest.config.js` など）は `.cjs` にリネームする。拡張子で CJS/ESM を判別するのが Node 22 の正式な運用法。

## 地雷②: ERR\_MODULE\_NOT\_FOUND — .js拡張子省略をsedで一括補完

Node 22 の ESM ローダーは拡張子省略を解決しない。TypeScript 上で書いた `import { foo } from './utils'` は `dist/` で `./utils.js` を見つけられずクラッシュする。

```bash
# エラー再現
node dist/main.js
# Error [ERR_MODULE_NOT_FOUND]: Cannot find module '.../dist/utils'
```

```bash
# src/ 配下の相対importに .js を一括付与 (GNU sed)
find src -name '*.ts' | xargs sed -i \
  "s/from '\(\.[^']*\)'/from '\1.js'/g"

# 二重付与を確認してから stage
git diff src/ | grep "^+" | grep "\.js\.js" && echo "CONFLICT" || echo "OK"
```

`tsc --noEmit` が通ったことを確認してから `git add` する。

## 地雷③: Named export not found — Vite 6 がバレルを破壊するパターン

Vite 6 の `optimizeDeps` がバレルファイル（`index.ts`）のツリーシェイクを行う際、循環依存があると名前付きエクスポートが消える。

```
[vite] Named export not found: 'ButtonGroup' from './components/index.ts'
```

```typescript
// NG: ワイルドカード再エクスポートは Vite 6 で壊れやすい
export * from './Button'
export * from './ButtonGroup'
```

```typescript
// OK: 明示的な named re-export に書き直す
export { Button } from './Button'
export { ButtonGroup } from './ButtonGroup'
```

循環依存の事前検出は `npx madge --circular src/` が速い。CI に組み込むと根本原因を量産しない。

## 地雷④⑤: \_\_dirname未定義 + top-level await禁止 — tsconfig 2行で同時解決

```
ReferenceError: __dirname is not defined in ES module scope
SyntaxError: Top-level await is not allowed in non-ECMAScript modules
```

この2つは tsconfig の `target` と `module` が古いままのときに同時に出る。

```jsonc
// tsconfig.json — Node 22 + ESM の最小正解設定
{
  "compilerOptions": {
    "target": "ES2022",            // top-level await を許可
    "module": "NodeNext",          // Node 22 ESM解決ルールに合わせる
    "moduleResolution": "NodeNext",
    "strict": true,
    "outDir": "dist"
  }
}
```

`__dirname` は ESM 非対応のため `import.meta.url` で代替する:

```typescript
// __dirname の ESM正式代替 (Node 22)
import { fileURLToPath } from 'node:url'
import { dirname, join } from 'node:path'

const __filename = fileURLToPath(import.meta.url)
const __dirname  = dirname(__filename)

const configPath = join(__dirname, '../config.json')  // 従来通り使える
```

## 第2章以降: 残り18エラーを Claude Code に90秒で診断させるプロンプト

上記5エラーはパターンが単純なため手動修正が現実的だった。ところが `paths` エイリアスと `resolve.alias` が衝突するケースや、`vite-plugin-dts` が生成した `.d.ts` が `NodeNext` 解決ルールと噛み合わないケースは、エラーと原因のリンクが直感では分からない。

本書の付録には以下のようなプロンプトテンプレートを5本収録している。貼るだけで Claude Code が unified diff 形式の修正パッチを出力する。

```
# 付録A-1: tsconfig/Vite設定エラー診断テンプレート
以下のスタックトレースとプロジェクト設定を診断してください。

## エラー
[スタックトレース全文をペースト]

## 設定ファイル
- package.json: [全文]
- tsconfig.json: [全文]
- vite.config.ts: [全文]

## 期待する出力
1. 根本原因を1行で
2. 修正差分を unified diff 形式で
3. CI で再発を防ぐ lint/type-check コマンドを1行で
```

第2章「tsconfig paths エイリアスと Vite resolve.alias の二重設定地雷」では、このテンプレートがどの精度でパッチを出すか、実測ログ付きで検証する。残り18種のエラーも同じパターンで解消できる。
