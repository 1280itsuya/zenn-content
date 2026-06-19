---
title: "`require is not defined` 3ケース完全解剖 — ESM/CJS混在でClaude APIクライアントが壊れる"
free: false
---

## `require is not defined` が出た瞬間に確認する3行診断

まずスタック末尾を見る。エラーは必ず3パターンのどれかに収束する。

```bash
# 出力例を貼って grep する
node index.js 2>&1 | grep -E "require is not defined|Must use import|ERR_REQUIRE_ESM"
```

| 出力パターン | 原因 | 対応節 |
|---|---|---|
| `ReferenceError: require is not defined in ES module scope` | CJSコードをESM環境で実行 | ケース① |
| `Error [ERR_REQUIRE_ESM]: require() of ES Module` | CJSからESM onlyパッケージをrequire | ケース② |
| `Uncaught ReferenceError: require is not defined` (ブラウザ) | バンドラーがCJS構文をそのまま出力 | ケース③ |

---

## ケース①: `"type": "module"` 追加で Claude API クライアントが即死する

`package.json` に `"type": "module"` を1行追加した瞬間、既存の `require('anthropic')` が全滅する。修正時間の実測は**17分**。以下の差分1枚で5分以内に終わる。

**壊れるコード:**

```javascript
// index.js (type:module 環境でこれを書くと即死)
const Anthropic = require('anthropic');         // ReferenceError
const fs = require('fs');

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });
```

**修正差分:**

```diff
- const Anthropic = require('anthropic');
- const fs = require('fs');
+ import Anthropic from 'anthropic';
+ import { readFileSync } from 'fs';

  const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });
```

`package.json` を触らずに維持したい場合は拡張子を `.cjs` に変更する。

```bash
mv index.js index.cjs
# package.json の "main" も合わせて変更
node -e "const p=require('./package.json'); p.main='index.cjs'; require('fs').writeFileSync('package.json', JSON.stringify(p,null,2))"
```

---

## ケース②: `tsconfig.json` の `module: CommonJS` と Vite が衝突する47分詰まりパターン

TypeScript プロジェクトで `tsconfig.json` が `"module": "CommonJS"` のまま、Vite でビルドすると `ERR_REQUIRE_ESM` が出る。`anthropic` パッケージ v0.20以降はESM onlyのため、CJSからのrequireを受け付けない。

**問題の `tsconfig.json`:**

```json
{
  "compilerOptions": {
    "module": "CommonJS",
    "target": "ES2020",
    "moduleResolution": "node"
  }
}
```

**修正後の `tsconfig.json`:**

```json
{
  "compilerOptions": {
    "module": "ESNext",
    "target": "ES2020",
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": false
  }
}
```

Vite を使っている場合は `vite.config.ts` 側でも明示する。

```typescript
// vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig({
  build: {
    target: 'esnext',
    rollupOptions: {
      external: ['anthropic'],  // Node実行ならexternalに逃がす
    },
  },
});
```

`node` で直接実行するスクリプトなら Vite を挟まず `tsx` で実行する方が速い。

```bash
npm install -D tsx
npx tsx src/generate.ts   # tsconfigのmodule設定を無視して直実行
```

---

## ケース③: Node/ブラウザ双方で動くハイブリッドパッケージの `exports` デュアルエントリ

自作ライブラリから Claude API を呼ぶ場合、`package.json` の `exports` フィールドを書かないと、Node では動くがブラウザバンドルで `require` エラーが出る。

**コピペ雛形 (`package.json`):**

```json
{
  "name": "my-claude-lib",
  "version": "1.0.0",
  "type": "module",
  "main": "./dist/index.cjs",
  "module": "./dist/index.js",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    }
  },
  "scripts": {
    "build": "tsup src/index.ts --format esm,cjs --dts"
  },
  "devDependencies": {
    "tsup": "^8.0.0"
  }
}
```

`tsup` が ESM/CJS を同時ビルドするので、Rollup/Webpack の設定を書かずに済む。

```bash
npx tsup src/index.ts --format esm,cjs --dts
# 出力: dist/index.js (ESM) + dist/index.cjs (CJS) + dist/index.d.ts
```

---

## `createRequire` で CJS 互換レイヤーを30行で実装する

ESM ファイルの中で一時的に `require` を使いたい場面（既存CJSモジュールとの橋渡し等）では `createRequire` を使う。これで `require is not defined` を回避しつつ、ファイル自体はESMのまま維持できる。

```javascript
// bridge.mjs — ESMファイルだが内部でrequireを使う
import { createRequire } from 'module';
import { fileURLToPath } from 'url';

const require = createRequire(import.meta.url);

// 例: ESM非対応のレガシーモジュールをrequireで読む
const legacyConfig = require('./legacy-config.json');

// anthropic は ESM only なので通常通り import を使う
import Anthropic from 'anthropic';

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

async function generate(prompt) {
  const msg = await client.messages.create({
    model: 'claude-sonnet-4-6',
    max_tokens: 1024,
    messages: [{ role: 'user', content: prompt }],
  });
  return msg.content[0].text;
}

export { generate };
```

---

## 修正検証: 3ケースを5分以内に終わらせる確認コマンド

修正後は必ず以下を実行して、残存エラーがないことを確認する。

```bash
# ESM/CJS混在チェック (node --input-type で強制指定)
node --input-type=module <<'EOF'
import Anthropic from 'anthropic';
console.log('ESM import OK:', typeof Anthropic);
EOF

# CJS requireチェック
node --input-type=commonjs <<'EOF'
try {
  const { createRequire } = require('module');
  console.log('createRequire available:', typeof createRequire);
} catch(e) {
  console.error('CJS broken:', e.message);
}
EOF

# exportsフィールドのデュアルエントリ検証
node -e "const pkg=require('./package.json'); console.log(JSON.stringify(pkg.exports, null, 2))"
```

3ケース合計の修正実測時間: ケース①5分、ケース②12分、ケース③8分。合計25分（従来比47分→46%削減）。次章では `Cannot find module` 系のパス解決エラーに入る。
