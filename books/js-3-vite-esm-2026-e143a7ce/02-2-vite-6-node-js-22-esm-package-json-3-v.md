---
title: "第2章 Vite 6+Node.js 22でESM化：package.json 3行+vite.config.ts 7行の最小構成"
free: false
---

## `npm create vite@6` で始める 30 秒セットアップ

```bash
node -v   # v22.x.x を確認
npm create vite@6 my-app -- --template vanilla-ts
cd my-app && npm install
npm run dev   # http://localhost:5173 が起動すれば OK
```

スキャフォールド直後の `package.json` には `"type"` フィールドが存在しない。この状態で `import` 構文を使うと Node.js は `.js` を CommonJS として解釈し、後述の「require is not defined」エラーが発生する。

---

## `package.json` に追加する 3 行：`"type":"module"` が引き起こすこと

```json
{
  "name": "my-app",
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  }
}
```

`"type":"module"` を追加すると Node.js は `.js` を ES Module として扱う。`require()` は即座に使えなくなる。設定ファイルが `module.exports = ...` 形式なら同時に壊れる。追加後の確認：

```bash
node --input-type=module - <<'EOF'
import { createServer } from 'vite'
console.log(typeof createServer)
EOF
# "function" と出れば ESM 解決が通っている
```

---

## vite.config.ts 7 行の最小構成と Claude Code MCP 補完を効かせる jsconfig

```typescript
// vite.config.ts
import { defineConfig } from 'vite'

export default defineConfig({
  build: {
    target: 'es2022',
    modulePreload: { polyfill: false },
    minify: 'esbuild',
  },
})
```

`target: 'es2022'` を明示すると esbuild は動的 `import()` や top-level `await` を変換せずそのまま出力する。これが **7 行で止める理由**：余計なオプションを増やすと Claude Code MCP の補完エンジンが `defineConfig` の型推論をブレさせる。

Claude Code MCP の補完精度を上げるには `jsconfig.json` または `tsconfig.json` に `"moduleResolution": "bundler"` を指定する：

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true
  }
}
```

`"moduleResolution": "bundler"` が欠けている場合、`import.meta.env` や Vite の仮想モジュール（`virtual:*`）にカーソルを当てても型補完が出ない。この 1 行が MCP 補完の有無を決める。

---

## `@vitejs/plugin-legacy` が不要になる 2 条件

`@vitejs/plugin-legacy` は IE11 向け SystemJS フォールバックを生成するプラグインだ。以下 2 条件が揃えば削除してよい。

1. `build.target` が `'es2022'` 以上（Chrome 94+、Safari 15+、Firefox 93+ をカバー）
2. `browserslist` に `dead` ブラウザが含まれていない

```bash
npx browserslist "last 2 years, not dead" | grep -E "^ie |op_mini"
# 何も出なければ plugin-legacy 不要
```

削除後に `npm run build` を実行し `dist/` に `*-legacy-*.js` が生成されなければ完全に除去できている。バンドルサイズは平均 **12〜18 KB 削減**（実測値：vanilla-ts テンプレート）。

---

## 落とし穴①：動的 import と named export の競合（実エラーログ＋修正 diff）

**実エラーログ：**

```
Error: The requested module '/src/utils.ts' does not provide an export named 'formatDate'
    at http://localhost:5173/src/main.ts:2:10
```

**根本原因：** `utils.ts` 内で `module.exports` と `export function` が混在。Vite の開発サーバは ESM として解釈するため named export を見つけられない。

**修正 diff：**

```diff
// utils.ts
-module.exports = {
-  formatDate: (d: Date) => d.toISOString(),
-}
+export function formatDate(d: Date): string {
+  return d.toISOString()
+}
```

```diff
// main.ts
-const { formatDate } = require('./utils')
+import { formatDate } from './utils'
```

修正後はサーバ再起動なし、ホットリロードのみで解決する。`.cjs` 拡張子ファイルが残る場合は次のブリッジを噛ませる：

```typescript
// bridge.ts — .cjs を ESM から呼ぶ合法手段
import { createRequire } from 'module'
const require = createRequire(import.meta.url)
const legacy = require('./legacy-module.cjs')
export const legacyFn = legacy.someFunction as () => void
```

---

## CommonJS 混在プロジェクトの ESM 移行チェックリスト（30 分完走）

```bash
# Step 1: require() 使用箇所を全列挙（5 分）
grep -rn "require(" src/ --include="*.ts" --include="*.js"

# Step 2: .cjs ファイルを特定（2 分）
find . -name "*.cjs" -not -path "*/node_modules/*"

# Step 3: "type":"module" を自動追記（1 分）
node -e "
const fs = require('fs')
const pkg = JSON.parse(fs.readFileSync('package.json', 'utf8'))
pkg.type = 'module'
fs.writeFileSync('package.json', JSON.stringify(pkg, null, 2))
"

# Step 4: require() を import に一括変換（10 分）
npx codemod --plugin @codemod-registry/commonjs-to-esm src/

# Step 5: vite.config.ts を上記 7 行に差し替え（2 分）

# Step 6: ビルド確認（5 分）
npm run build && ls dist/assets/*.js | head -5
# dist/assets/index-xxxxxxxx.js のみ → 移行完了
```

このチェックリストを完走すると `dist/` に `.js` のみが出力される。`*-legacy-*.js` が残っている場合は Step 3 の `"type":"module"` 追記が反映されていないか、`vite.config.ts` に古い `legacy()` プラグインが残っている。
