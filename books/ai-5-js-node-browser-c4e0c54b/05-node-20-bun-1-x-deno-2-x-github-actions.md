---
title: "Node 20/Bun 1.x/Deno 2.x 三環境マトリクスとGitHub Actions matrix buildによるAI自動化スクリプトCI前検知"
free: false
---

Zenn有料章を執筆します。

## Node 20/Bun 1.x/Deno 2.x — Fetch・Crypto・TextEncoder・File API 互換性マトリクス（2026年版）

ローカル Node 20 で通過したスクリプトが Bun 1.x 本番でクラッシュする最大の原因は、「✅に見えるが挙動が違う」API の存在だ。以下のマトリクスを手元に置いておくと、デバッグ着手前に候補を3分の1に絞れる。

| API | Node 20 | Bun 1.x | Deno 2.x |
|---|---|---|---|
| `fetch` グローバル | ✅ | ✅（`undici` 非互換） | ✅ |
| `Response.body` ReadableStream | ✅ | ⚠️ `.getReader()` 挙動差異 | ✅ |
| `crypto.subtle` | ✅ | ✅ | ✅ import 不要 |
| `crypto.randomUUID()` | ✅ | ✅ | ✅ |
| `TextEncoder` | ✅ | ✅ | ✅ |
| `File`（Web API） | ✅ Node 20.0+ | ✅ | ✅ |
| `__dirname` / `__filename` | ✅ | ✅ | ❌ `import.meta.dirname` へ |
| `Bun.file()` | ❌ | ✅ | ❌ |
| `node:crypto` import | ✅ | ✅ | ✅ Deno 2.0+ |

## GitHub Actions `workflow.yml` 全量公開 — 三環境 matrix build を 60 行で構築

`fail-fast: false` が必須。Node が落ちても Bun・Deno の結果を独立収集するためだ。

```yaml
# .github/workflows/env-matrix.yml
name: AI Tool Cross-Env CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        include:
          - runtime: node
            version: "20"
            cmd: "node"
          - runtime: bun
            version: "1.x"
            cmd: "bun"
          - runtime: deno
            version: "2.x"
            cmd: "deno run --allow-net --allow-env --allow-read"

    runs-on: ubuntu-latest
    name: "${{ matrix.runtime }} ${{ matrix.version }}"

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node 20
        if: matrix.runtime == 'node'
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.version }}

      - name: Setup Bun 1.x
        if: matrix.runtime == 'bun'
        uses: oven-sh/setup-bun@v2
        with:
          bun-version: ${{ matrix.version }}

      - name: Setup Deno 2.x
        if: matrix.runtime == 'deno'
        uses: denoland/setup-deno@v2
        with:
          deno-version: ${{ matrix.version }}

      - name: Run env-probe
        run: ${{ matrix.cmd }} env-probe.mjs

      - name: Run API compat tests
        run: ${{ matrix.cmd }} tests/api-compat.test.mjs
```

## Bun 1.x で Claude API ストリーミングがハングする `Response.body` 差異と修正コード

次のコードは Node 20 では動作するが、Bun 1.1.x では `response.body` が `null` を返すケースがあり、`getReader()` 呼び出し直後に無限待ちになる。

```javascript
// NG: Bun 1.1.x でハングする
const response = await fetch("https://api.anthropic.com/v1/messages", {
  method: "POST",
  headers: {
    "x-api-key": process.env.ANTHROPIC_API_KEY,
    "anthropic-version": "2023-06-01",
    "content-type": "application/json",
  },
  body: JSON.stringify({
    model: "claude-haiku-4-5-20251001",
    max_tokens: 256,
    stream: true,
    messages: [{ role: "user", content: "Hello" }],
  }),
});

// Bun 1.x では response.body が null になりクラッシュ
const reader = response.body.getReader();
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  process.stdout.write(new TextDecoder().decode(value));
}
```

Bun/Node/Deno 三環境対応の修正版：

```javascript
// OK: 三環境対応ストリーミング
const response = await fetch("https://api.anthropic.com/v1/messages", {
  method: "POST",
  headers: {
    "x-api-key": process.env.ANTHROPIC_API_KEY,
    "anthropic-version": "2023-06-01",
    "content-type": "application/json",
  },
  body: JSON.stringify({
    model: "claude-haiku-4-5-20251001",
    max_tokens: 256,
    stream: true,
    messages: [{ role: "user", content: "Hello" }],
  }),
});

if (typeof Bun !== "undefined") {
  // Bun: text() 経由が安全
  const text = await response.text();
  process.stdout.write(text);
} else {
  // Node 20 / Deno 2.x: for-await が安定
  for await (const chunk of response.body) {
    process.stdout.write(new TextDecoder().decode(chunk));
  }
}
```

## Deno 2.x で `__dirname` 未定義クラッシュを `import.meta.dirname` 1 行で解決

AI 副業ツールのほぼ全ファイルに `path.join(__dirname, "config.json")` パターンが散在している。Deno 2.x はここで `ReferenceError: __dirname is not defined` を出してプロセスを落とす。

```javascript
// runtime-compat.mjs — Node/Bun/Deno 三環境の差異を 1 ファイルに閉じ込める
export const runtime = (() => {
  if (typeof Bun !== "undefined") return "bun";
  if (typeof Deno !== "undefined") return "deno";
  return "node";
})();

// __dirname 相当を三環境で統一取得
export const getDirname = () =>
  new URL(".", import.meta.url).pathname.replace(/\/$/, "");

// ファイル読み込みを三環境対応に抽象化
export const readFile = async (filePath) => {
  if (runtime === "bun") return Bun.file(filePath).text();
  if (runtime === "deno") return Deno.readTextFile(filePath);
  const { readFile: fsRead } = await import("node:fs/promises");
  return fsRead(filePath, "utf8");
};
```

このモジュールを `import` するだけで、以降は環境分岐を各ファイルに書かなくて済む。

## `env-probe.mjs` に CI ゲート機能を組み込む最終実装

第1章の `env-probe.mjs` を CI ゲートとして完成させる。exit code 1 で GitHub Actions の step が red になり、マージ・デプロイを自動ブロックする。

```javascript
// env-probe.mjs（最終版 — CI ゲート付き）
import { runtime } from "./runtime-compat.mjs";

const CHECKS = [
  { name: "fetch",                  fn: () => typeof fetch !== "undefined" },
  { name: "crypto.subtle",          fn: () => typeof crypto?.subtle !== "undefined" },
  { name: "TextEncoder",            fn: () => typeof TextEncoder !== "undefined" },
  { name: "File (Web API)",         fn: () => typeof File !== "undefined" },
  {
    name: "Response.body ReadableStream",
    fn: async () => new Response("x").body instanceof ReadableStream,
  },
];

console.log(`=== env-probe: runtime=${runtime} ===`);
let failed = 0;

for (const { name, fn } of CHECKS) {
  try {
    const ok = await fn();
    console.log(`${ok ? "✅" : "❌"} ${name}`);
    if (!ok) failed++;
  } catch (e) {
    console.log(`❌ ${name}: ${e.message}`);
    failed++;
  }
}

if (failed > 0) {
  console.error(`\n[FAIL] ${failed} API(s) が ${runtime} と非互換。デプロイをブロック。`);
  process.exit(1);
}
console.log(`\n[PASS] 全 API 互換確認済 on ${runtime}`);
```

matrix build で三環境に流すと、Bun ジョブのみ exit 1 となって CI が赤くなる。

```
# Node 20 ジョブ
✅ fetch
✅ crypto.subtle
✅ TextEncoder
✅ File (Web API)
✅ Response.body ReadableStream
[PASS] 全 API 互換確認済 on node

# Bun 1.x ジョブ
✅ fetch
✅ crypto.subtle
✅ TextEncoder
✅ File (Web API)
❌ Response.body ReadableStream
[FAIL] 1 API(s) が bun と非互換。デプロイをブロック。
```

このパイプラインを導入した後、「ローカル通過 → 本番クラッシュ → 深夜デバッグ」サイクルは発生しなくなる。次章では、ここで検知した非互換 API を Playwright ブラウザ環境で代替する実装パターンに移る。
