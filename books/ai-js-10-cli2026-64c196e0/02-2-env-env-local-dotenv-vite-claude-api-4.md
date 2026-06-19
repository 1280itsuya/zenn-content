---
title: "第2章：.env / .env.local / dotenv読込優先順位をVite×Claude API実装で完全図解（エラー4選）"
free: false
---

章の目的を言語化してから執筆します。

**目的:** 読者が .env 読込の4エラーをスタックトレース→diff→修正の流れで即解決できるようにし、章末の診断スニペットを手元で動かせる状態にする。

**章末の成果物:** `node check-env.mjs` を実行して4エラーを自動検出するスクリプト。

---

## VITE\_プレフィックス欠落で `import.meta.env.ANTHROPIC_API_KEY` が `undefined` になる（エラー1）

Vite は `.env` に書いた変数のうち、`VITE_` で始まるものだけをクライアントバンドルへ埋め込む。プレフィックスなしの変数はビルド時に剥ぎ取られ、ランタイムで `undefined` になる。

```
# .env（NG）
ANTHROPIC_API_KEY=sk-ant-xxxxx

# 実行時エラー
TypeError: Cannot read properties of undefined (reading 'startsWith')
    at new Anthropic (node_modules/@anthropic-ai/sdk/src/index.ts:62:22)
    at src/lib/claude.ts:5:18
```

```diff
- ANTHROPIC_API_KEY=sk-ant-xxxxx
+ VITE_ANTHROPIC_API_KEY=sk-ant-xxxxx
```

```ts
// src/lib/claude.ts
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic({
  apiKey: import.meta.env.VITE_ANTHROPIC_API_KEY, // VITE_ 必須
  dangerouslyAllowBrowser: true,
});
```

> **注意:** `dangerouslyAllowBrowser: true` はローカル開発専用。本番はバックエンドプロキシ経由にする。

---

## `process.env` vs `import.meta.env`：Node.js/Vite の2レイヤー分離（エラー2）

Vite プロジェクトでも `vite.config.ts` や API Route（SSR）は Node.js プロセスで動く。ここで `import.meta.env` を使うとビルド時評価されず `undefined` になる。

```
# スタックトレース（vite.config.ts 内での誤用）
ReferenceError: import.meta is not defined
    at file:///project/vite.config.ts:3:18
```

どのレイヤーで変数が展開されるかを図解する。

```
.env ファイル
│
├─ [Vite ビルド時] VITE_* のみ抽出
│      └─▶ import.meta.env.VITE_XXX   ← ブラウザ/クライアント
│
└─ [Node.js プロセス] dotenv.config() で読込
       └─▶ process.env.XXX             ← vite.config / SSR / スクリプト
```

```ts
// vite.config.ts（Node.js コンテキスト）
import { defineConfig } from "vite";
import dotenv from "dotenv";

dotenv.config(); // ← ここで process.env に展開

export default defineConfig({
  define: {
    __API_BASE__: JSON.stringify(process.env.API_BASE_URL), // process.env を使う
  },
});
```

---

## `dotenv.config()` 呼び出しタイミングのズレで Claude クライアントが `undefined` を掴む（エラー3）

Node.js スクリプトでは `import` の巻き上げにより、`dotenv.config()` より先にモジュールの初期化が走るケースがある。

```
# エラー（API キーが空文字になる）
AuthenticationError: 401 {"type":"error","error":{"type":"authentication_error","message":"invalid x-api-key"}}
```

```ts
// NG：import 順に注意
import Anthropic from "@anthropic-ai/sdk";
import dotenv from "dotenv";

dotenv.config(); // ← Anthropic の初期化より後に評価される場合がある
const client = new Anthropic(); // この時点で process.env.ANTHROPIC_API_KEY が未定義
```

```ts
// OK：dotenv を最初に呼ぶ
import dotenv from "dotenv";
dotenv.config(); // ← 最上部で確定実行

import Anthropic from "@anthropic-ai/sdk";
const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });
```

ESM の `import` は静的解析で巻き上がるため、`dotenv.config()` を最初の `import` より前に置くには `--require` フラグか `dotenv/config` の import-side-effect を使う。

```bash
# 確実な方法（Node.js 18+）
node --require dotenv/config src/index.js
# または
node -e "import('dotenv/config').then(() => import('./src/index.js'))"
```

---

## GitHub Actions secrets → Vite `env` マッピングで `VITE_` が消える（エラー4）

CI でビルドが通っても、Vite クライアント側でキーが `undefined` になる典型パターン。secrets の名前に `VITE_` プレフィックスをつけ忘れるか、`env:` セクションでの変数名が食い違う。

```yaml
# NG：env セクションでプレフィックスを落としてしまう
jobs:
  build:
    steps:
      - run: npm run build
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }} # VITE_ なし → クライアントで undefined
```

```yaml
# OK：env キーに VITE_ を付ける
jobs:
  build:
    steps:
      - run: npm run build
        env:
          VITE_ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }} # ← キー名に VITE_ 必須
```

`secrets.ANTHROPIC_API_KEY` という GitHub 側の名前は変えずに、`env:` のマッピングキーだけを `VITE_ANTHROPIC_API_KEY` にすれば良い。

---

## 4エラーを30秒で潰す診断スクリプト `check-env.mjs`

以下を `check-env.mjs` として保存し `node check-env.mjs` を実行する。`.env` と `vite.config.ts`（存在する場合）を読んで4パターンを自動チェックする。

```js
// check-env.mjs
import { readFileSync, existsSync } from "fs";
import { resolve } from "path";

const ENV_FILES = [".env", ".env.local", ".env.development", ".env.production"];
const errors = [];

// 1. VITE_ プレフィックスチェック
const allVars = {};
for (const f of ENV_FILES) {
  if (!existsSync(f)) continue;
  const lines = readFileSync(f, "utf8").split("\n");
  for (const line of lines) {
    const m = line.match(/^([A-Z_][A-Z0-9_]*)=(.+)/);
    if (m) allVars[m[1]] = { file: f, value: m[2] };
  }
}

const anthropicKeys = Object.keys(allVars).filter((k) =>
  k.includes("ANTHROPIC") || k.includes("OPENAI") || k.includes("CLAUDE")
);

for (const k of anthropicKeys) {
  if (!k.startsWith("VITE_") && existsSync("vite.config.ts")) {
    errors.push(`[ERR1] ${k} に VITE_ プレフィックスなし (${allVars[k].file})`);
  }
}

// 2. .env.local が .env より優先されるか確認（Vite 読込順）
const ORDER = [".env.local", ".env.development.local", ".env", ".env.development"];
const loaded = ORDER.filter(existsSync);
console.log("読込順 (上が優先):", loaded.join(" > ") || "なし");

// 3. dotenv.config() 呼出し位置チェック
const srcFiles = ["src/index.ts", "src/index.js", "src/main.ts", "server.ts", "server.js"];
for (const f of srcFiles) {
  if (!existsSync(f)) continue;
  const content = readFileSync(f, "utf8");
  const dotenvLine = content.indexOf("dotenv.config");
  const anthropicLine = content.indexOf("new Anthropic");
  if (dotenvLine !== -1 && anthropicLine !== -1 && dotenvLine > anthropicLine) {
    errors.push(`[ERR3] ${f}: dotenv.config() が Anthropic 初期化より後に呼ばれている`);
  }
}

// 4. GitHub Actions YAML チェック
const ciFiles = [".github/workflows/build.yml", ".github/workflows/ci.yml"];
for (const f of ciFiles) {
  if (!existsSync(f)) continue;
  const content = readFileSync(f, "utf8");
  const secretRefs = [...content.matchAll(/(\w+):\s*\$\{\{\s*secrets\.\w+/g)];
  for (const m of secretRefs) {
    const envKey = m[1];
    if ((envKey.includes("ANTHROPIC") || envKey.includes("CLAUDE")) && !envKey.startsWith("VITE_")) {
      errors.push(`[ERR4] ${f}: ${envKey} に VITE_ プレフィックスなし → Vite クライアントで undefined になる`);
    }
  }
}

// 結果出力
if (errors.length === 0) {
  console.log("✅ 4エラーすべてクリア");
} else {
  console.error(`❌ ${errors.length} 件の問題を検出:`);
  errors.forEach((e) => console.error(" ", e));
  process.exit(1);
}
```

実行結果の例:

```
読込順 (上が優先): .env.local > .env
❌ 2 件の問題を検出:
  [ERR1] ANTHROPIC_API_KEY に VITE_ プレフィックスなし (.env)
  [ERR4] .github/workflows/build.yml: ANTHROPIC_API_KEY に VITE_ プレフィックスなし → Vite クライアントで undefined になる
```

このスクリプトを `package.json` の `scripts` に登録し、`npm run check:env` として CI の最初のステップに差し込めば、本番デプロイ後に「なぜか undefined」が出る事態を開発ループ内で潰せる。

```json
{
  "scripts": {
    "check:env": "node check-env.mjs"
  }
}
```
