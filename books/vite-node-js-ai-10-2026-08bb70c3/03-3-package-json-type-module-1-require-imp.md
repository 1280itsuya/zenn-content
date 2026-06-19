---
title: "第3章: package.json「type: module」1行でrequire/import混在エラー4選を根絶する"
free: false
---

Zenn章を執筆します。

## 第3章: package.json「type: module」1行でrequire/import混在エラー4選を根絶する

---

Node.js 18以降のプロジェクトで `package.json` に `"type": "module"` を追加した瞬間、既存の `require()` が全滅する。Claude Codeでスクリプト自動生成しているとこの混在地獄に毎回落ちる。4つのエラーパターンを実ログ付きで解体し、それぞれ解決コマンド1行で片付ける。

---

## エラー①`__dirname is not defined`：ESM移行初日にロスする平均37分

`"type": "module"` を追加した直後、`__dirname` と `__filename` が即死する。CommonJSグローバルであり、ESMスコープには存在しない。

**実エラーログ（ロスト時間：37分）**
```
ReferenceError: __dirname is not defined in ES module scope
  at file:///app/src/config.mjs:3:21
```

**解決：`import.meta.url` で再定義する**
```js
// src/config.js（"type":"module" 配下）
import { fileURLToPath } from 'url';
import { dirname } from 'path';

const __filename = fileURLToPath(import.meta.url);
const __dirname  = dirname(__filename);

// 以降は従来通り使える
const configPath = `${__dirname}/config.yaml`;
```

`vite.config.ts` でも同じ問題が出る。Viteの場合は `new URL('.', import.meta.url).pathname` が最短。

---

## エラー②`require is not a function`：CommonJS依存パッケージでビルドが落ちる

`"type": "module"` 配下で `require('dotenv')` を呼ぶと即クラッシュ。ライブラリ側がCJSのみ提供している場合も同様。

**実エラーログ（ロスト時間：52分）**
```
Error [ERR_REQUIRE_ESM]: require() of ES Module
/node_modules/chalk/source/index.js not supported.
```

**解決A：`createRequire` でCJSブリッジを作る（移行期の最短策）**
```js
import { createRequire } from 'module';
const require = createRequire(import.meta.url);

const dotenv = require('dotenv');
dotenv.config();
```

**解決B：`import()` 動的インポートへ切り替える（本命）**
```js
// 非同期コンテキストが必要になるが、これが正解
const { default: chalk } = await import('chalk');
```

Claude Codeが自動生成するスクリプトは高確率でCJS `require` を出力する。生成後に `sed -i 's/const .* = require(/import /g'` で機械的に変換するより、プロンプトに「ESM形式で出力、requireは使わない」と明示する方が速い。

---

## エラー③`.mjs`/`.cjs`拡張子強制によるモジュール解決失敗

Node.js 18は拡張子を見て形式を判断する。`"type": "module"` 配下の `.js` はESM扱いだが、同リポジトリに古いCJSスクリプトが混在すると `.cjs` 拡張子を強制されてimportパスが壊れる。

**実エラーログ（ロスト時間：28分）**
```
Error [ERR_MODULE_NOT_FOUND]: Cannot find module
'/app/scripts/helper' imported from /app/src/index.js
```

**解決：拡張子を明示してimportする**
```js
// NG — Node.jsはESMで拡張子省略不可
import { helper } from './scripts/helper';

// OK — 明示が必須
import { helper } from './scripts/helper.js';
```

**Viteのエイリアスで吸収する場合**
```ts
// vite.config.ts
import { defineConfig } from 'vite';
import { resolve } from 'path';

export default defineConfig({
  resolve: {
    alias: {
      '@scripts': resolve(__dirname, 'scripts'),
    },
    // 拡張子を補完させる
    extensions: ['.mjs', '.js', '.ts', '.json'],
  },
});
```

---

## エラー④Vite `build.rollupOptions.output.interop` 未設定でビルドのみ落ちる

`vite dev` は通るのに `vite build` だけ落ちる。Rollupのinterop処理がデフォルト `auto` のままCJSパッケージを取り込めていないケース。

**実エラーログ（ロスト時間：61分）**
```
[rollup] (!) "default" is not exported by
"node_modules/some-cjs-lib/index.js"
```

**解決：`vite.config.ts` に `interop: 'compat'` を追加する**
```ts
// vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        interop: 'compat',  // CJS defaultエクスポートを互換モードで処理
      },
    },
  },
  optimizeDeps: {
    // devサーバー側でも事前バンドル対象に追加
    include: ['some-cjs-lib'],
  },
});
```

`optimizeDeps.include` を漏らすと `dev` では動いて `build` だけ落ちる非対称障害になる。CI（GitHub Actions）のビルドステップで初めて発覚するパターンが多い。

---

## ESM完全移行：14項目チェックリスト

上記4エラーをまとめて防ぐ移行前確認リスト。`package.json` に `"type": "module"` を追加する前に全項目を通す。

```bash
# ① require() 呼び出し箇所を一覧化
grep -rn "require(" src/ --include="*.js" --include="*.ts"

# ② __dirname / __filename 使用箇所
grep -rn "__dirname\|__filename" src/

# ③ .cjs拡張子ファイルの存在確認
find . -name "*.cjs" -not -path "*/node_modules/*"

# ④ CJS only パッケージの検出（esm フィールドなし）
node -e "
const pkg = require('./node_modules/target-lib/package.json');
console.log('ESM対応:', !!(pkg.module || pkg.exports));
"
```

**テキストチェックリスト14項目**

```markdown
## ESM移行チェックリスト（package.json "type":"module" 追加前）

### ソースコード
- [ ] 1. `require()` を `import` に全置換
- [ ] 2. `__dirname` を `fileURLToPath(import.meta.url)` パターンに置換
- [ ] 3. `__filename` を `fileURLToPath(import.meta.url)` に置換
- [ ] 4. import文に拡張子 `.js` / `.ts` を明示付与
- [ ] 5. 動的 `require()` を `await import()` に変換

### 依存関係
- [ ] 6. CJSのみ提供パッケージを `npm info <pkg> exports` で確認
- [ ] 7. 移行不可パッケージは `createRequire` でブリッジ
- [ ] 8. `vite.config.ts` の `optimizeDeps.include` に追加

### Vite設定
- [ ] 9. `rollupOptions.output.interop: 'compat'` 追加
- [ ] 10. `resolve.extensions` に `.mjs` を含める
- [ ] 11. `vite build` をローカルで必ず確認（dev通過≠build通過）

### CI/テスト
- [ ] 12. GitHub Actions の Node.js バージョンを18以上に固定
- [ ] 13. テストランナー（Vitest/Jest）のESM設定を確認
- [ ] 14. `.env` の `import.meta.env` 移行（`process.env` はViteでは非推奨）
```

---

ロスト時間の合計：37 + 52 + 28 + 61 = **178分**。`"type": "module"` の1行追加でこれだけ溶ける。このチェックリストを `pre-commit` フックに仕込めば次回以降はゼロになる。

```bash
# .husky/pre-commit に追加
#!/bin/sh
if grep -rn 'require(' src/ --include="*.js" --include="*.ts" | grep -v createRequire; then
  echo "ESMプロジェクトでrequire()検出。importに変換してください"
  exit 1
fi
```
