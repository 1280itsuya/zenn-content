---
title: "初日に99%が踏む3大エラー実録──ERR_REQUIRE_ESM・.env未読み・monorepo落とし穴"
free: true
---

## ERR_REQUIRE_ESM ── `@anthropic-ai/sdk` を `require()` で読んだ瞬間に落ちる

Vite プロジェクトに Claude SDK を追加した翌朝、こんなスタックトレースが出る。

```
Error [ERR_REQUIRE_ESM]: require() of ES Module
/node_modules/@anthropic-ai/sdk/index.js not supported.
Instead change the require of index.js to a dynamic import()
which is available in all CommonJS modules.
    at Object.<anonymous> (/scripts/generate.js:1:18)
```

`@anthropic-ai/sdk` v0.24 以降は `"type": "module"` のピュア ESM パッケージだ。`require()` では読めない。

**1行修正：**

```diff
- const Anthropic = require('@anthropic-ai/sdk');
+ import Anthropic from '@anthropic-ai/sdk';
```

ただしこれだけでは足りない。`package.json` に `"type": "module"` がなければ Node.js がファイルを CJS として解釈し続ける。

```json
// package.json に追加
{
  "type": "module"
}
```

既存の `require()` が他にあれば `import` に一括置換する。`__dirname` は ESM では使えないので下記に差し替える。

```js
// ESM で __dirname 相当を得る
import { fileURLToPath } from 'url';
import { dirname } from 'path';
const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
```

---

## `import.meta.env` が本番で `undefined` ── Vite の `.env` 優先順位の罠

Vite は `.env.local` → `.env` の順で読む。ところが `dotenv-cli` 経由でスクリプトを起動すると、**Vite のビルドパイプラインを経由しない Node.js プロセス**に環境変数を渡すことになり、`import.meta.env.VITE_CLAUDE_API_KEY` は `undefined` になる。

実際のエラーログ（`console.log` で発覚するタイプ）：

```
VITE_CLAUDE_API_KEY: undefined
AnthropicError: API key is required
```

**根本原因：** `import.meta.env` は Vite がバンドル時に静的置換する仕組みで、Node.js ランタイムには存在しない。

**修正パターン（Node.js スクリプト側）：**

```js
// scripts/generate.mjs
import 'dotenv/config'; // .env を Node プロセスに読ませる
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic({ apiKey: process.env.CLAUDE_API_KEY });
```

```ini
# .env（VITE_ プレフィックスは Vite フロント専用）
CLAUDE_API_KEY=sk-ant-xxxxxxxxxxxx
```

フロントエンドに渡す値は `VITE_` プレフィックス、Node.js スクリプトに渡す値はプレフィックスなし、と分離するのが正解だ。

---

## monorepo で `Cannot find package 'playwright'` ── hoisting 落とし穴

`pnpm workspaces` や `npm workspaces` の monorepo で Playwright を子パッケージに入れると、ルートの `node_modules` に hoist されずこのエラーが出る。

```
Error: Cannot find package 'playwright' imported from
/packages/scraper/src/index.mjs
```

`pnpm` は意図的に hoisting を制限する。`npm` でも `--workspaces` 環境によって挙動が変わる。

**修正：`.npmrc` に hoisting 設定を追加（pnpm）：**

```ini
# .npmrc（ルート）
public-hoist-pattern[]=*playwright*
public-hoist-pattern[]=*@playwright*
```

```bash
# 設定後に再インストール
pnpm install
pnpm --filter scraper exec playwright install chromium
```

あるいはルートの `package.json` に `playwright` を devDependencies として追加し、子パッケージからは参照だけさせる方法もある。

```jsonc
// ルート package.json
{
  "devDependencies": {
    "playwright": "^1.44.0"
  }
}
```

---

## 3エラーの共通構造 ── 「AI ツール固有の依存衝突」という視点

3つのエラーを並べると、すべて同じ原因に収束する。

| エラー | 発生場所 | 根本 |
|---|---|---|
| ERR_REQUIRE_ESM | Node.js ランタイム | CJS/ESM の境界 |
| import.meta.env undefined | Vite バンドル境界 | ランタイムとビルダーの混同 |
| Cannot find package | monorepo resolver | hoisting ポリシーの差異 |

いずれも「汎用の Node.js 入門書が想定しないレイヤー」で起きる。`@anthropic-ai/sdk` `playwright` `dotenv-cli` が ESM-first で設計されているため、2020年以前のCJS前提のプロジェクト構成とそのまま組み合わせると必ず衝突する。

**第2章以降で扱う残り7エラー：**

- `vite.config.ts` での `__dirname` 参照エラー
- `tsconfig.json` の `moduleResolution` 不一致によるパス解決失敗
- `playwright` の `browser.close()` 未呼び出しによるプロセスリーク
- `dotenv-cli` と `cross-env` の実行順序バグ
- `pnpm` の peer dependency strict モードによるインストール失敗
- GitHub Actions 上での `PLAYWRIGHT_BROWSERS_PATH` 未設定
- Claude API の `stream: true` レスポンスを `JSON.parse` に渡すランタイムエラー

これら7件はすべて「実プロジェクトで実際に踏んだログ」と修正 diff 付きで解説する。第1章の3エラーが全部既視感なら、残りも確実に踏む。
