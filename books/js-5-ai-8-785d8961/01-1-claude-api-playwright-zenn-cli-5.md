---
title: "第1章（無料）：Claude API・Playwright・zenn-cliが死ぬ5エラーの実物スタックトレースと損失時間レポート"
free: true
---

## 第1章（無料）：Claude API・Playwright・zenn-cliが死ぬ5エラーの実物スタックトレースと損失時間レポート

AI副業ツールを実務投入した初週に、これら5パターンのエラーで合計**31時間・¥2,340相当のAPI費用**を溶かした。原因はすべてJSのモジュール設定1行だった。

---

## 5エラーで失った時間とAPI費用の実測値

| エラー | 詰まり時間 | 無駄になったAPI費用 |
|--------|-----------|------------------|
| Cannot use import statement | 14時間 | ¥1,120 |
| require is not defined | 3時間 | ¥240 |
| fetch is not a function | 6時間 | ¥480 |
| localStorage is not defined | 4時間 | ¥320 |
| \_\_dirname is not defined | 4時間 | ¥180 |

「API費用」はエラー再現・デバッグ中に流したClaudeへのリクエスト費用。¥100前後でも、毎日踏めば月¥3,000超える。

---

## エラー1：Claude APIスクリプトで踏む「Cannot use import statement」

```
/project/claude-bot.js:1
import Anthropic from "@anthropic-ai/sdk";
^^^^^^

SyntaxError: Cannot use import statement in module
    at wrapSafe (node:internal/modules/cjs/loader:1308:18)
    at Module._compile (node:internal/modules/cjs/loader:1357:20)
    at Object.Module._extensions..js (node:internal/modules/cjs/loader:1412:10)
```

**発生環境**：Node 18 + `package.json` に `"type": "module"` なし + `@anthropic-ai/sdk` v0.20以降（ESM専用export）

根本原因：SDKがESM専用なのに、ファイルをCJSとして実行している。

---

## エラー2：zenn-cliビルド時の「require is not defined」

```
file:///project/scripts/publish.mjs:3:18
const path = require("path");
             ^

ReferenceError: require is not defined in ES module scope, you can use import instead
    at file:///project/scripts/publish.mjs:3:18
    at ModuleJob.run (node:internal/modules/esm/module_job:218:25)
```

**発生環境**：`zenn-cli` のビルドスクリプトを `.mjs` で書いた + 古い `require()` 構文を混在

zenn-cli自体はCJS互換だが、呼び出しスクリプトをESMにした瞬間に `require` が死ぬ。

---

## エラー3：Node 16以前 + Playwright並走時の「fetch is not a function」

```
TypeError: fetch is not a function
    at callClaudeAPI (/project/src/claude-client.js:22:18)
    at async Page.<anonymous> (/project/tests/e2e.spec.ts:45:5)
    at async Promise.all (index 0)

Node.js v16.20.2
```

**発生環境**：Playwright 1.40 + Node 16（`fetch` はNode 18からグローバル）+ `node-fetch` 未インストール

```bash
node --version   # v16.20.2
# fetch はグローバルに存在しない
```

---

## エラー4：Playwright SSR環境の「localStorage is not defined」

```
ReferenceError: localStorage is not defined
    at Object.<anonymous> (/project/src/auth-helper.js:8:3)
    at Module._compile (node:internal/modules/cjs/loader:1357:20)
    ...
    at /project/node_modules/@playwright/test/lib/runner/runner.js:444:11

# このスクリプトを呼んだAPI費用: ¥320（エラーでも課金は走った）
```

**発生環境**：PlaywrightのNodeプロセス（サーバーサイド）で `localStorage` にアクセス

ブラウザAPIをPlaywrightのsetup/teardownスクリプトに書くと必ず踏む。

---

## エラー5：zenn-cli投稿スクリプトの「\_\_dirname is not defined」

```
file:///project/scripts/upload-images.mjs:5:16
const dir = __dirname;
            ^^^^^^^^^

ReferenceError: __dirname is not defined in ES module scope
    at file:///project/scripts/upload-images.mjs:5:16
```

**発生環境**：`package.json` に `"type": "module"` あり + `__dirname` 使用

---

## 自分のエラーを30秒で特定する診断チェックリスト

```
□ エラーメッセージに "import statement" → パターン1（CJS/ESM混在）
□ エラーメッセージに "require is not defined" → パターン2（.mjsでrequire使用）
□ エラーメッセージに "fetch is not a function" → パターン3（Node 16以下）
□ エラーメッセージに "localStorage is not defined" → パターン4（Node環境でBrowser API）
□ エラーメッセージに "__dirname is not defined" → パターン5（ESMで__dirname使用）
```

パターンが特定できたなら、第2章以降に**各パターンの根本原因1行＋コピペで動く修正パッチ**を全掲載している。14時間かかった修正が、パッチ1枚で3分に短縮される。
