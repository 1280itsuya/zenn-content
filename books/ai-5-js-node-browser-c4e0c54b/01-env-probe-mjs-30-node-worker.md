---
title: "`env-probe.mjs` 30行でNode/ブラウザ/Worker環境を自動判定する診断キット実装【無料試し読み】"
free: true
---

```markdown
AI副業ツールが動かない根本原因の8割は、実行環境の誤認識だ。Node.js向けコードをブラウザで走らせる、Cloudflare Workersに`process.env`を直書きする、Denoで`require()`を呼ぶ——エラーメッセージは異なるが出発点は同じ。`env-probe.mjs`を30行で実装し、Claude APIクライアントを初期化する前のチェックポイントとして組み込む手順を示す。

## `typeof window` / `process.versions` / `globalThis` — 3つのAPIで4環境を一意に特定する

`typeof window !== 'undefined'`が`true`ならブラウザかServiceWorker、`false`かつ`process.versions.node`が存在するならNode.jsと断定できる。Cloudflare Workersは`typeof window === 'undefined'`かつ`typeof caches !== 'undefined'`という組み合わせで識別し、Denoは`typeof Deno !== 'undefined'`で一発確定だ。この3点セットで主要4環境を網羅する。

```javascript
// 各ランタイムで試せる最速ワンライナー (DevTools / Node REPL / Deno REPL に貼る)
console.log({
  hasWindow:  typeof window  !== 'undefined',
  hasProcess: typeof process !== 'undefined' && !!process.versions?.node,
  hasCaches:  typeof caches  !== 'undefined',
  hasDeno:    typeof Deno    !== 'undefined',
});
```

ブラウザのDevToolsコンソール、Node.js REPL、`wrangler dev`の3環境でそれぞれ実行すると出力が変わる。この差が診断の基礎になる。

## `env-probe.mjs` 30行全文 — Node / ブラウザ / Cloudflare Workers / Deno を自動分類

```javascript
// env-probe.mjs
export const probe = () => {
  const flags = {
    hasWindow:  typeof window  !== 'undefined',
    hasProcess: typeof process !== 'undefined' && !!process.versions?.node,
    hasCaches:  typeof caches  !== 'undefined',
    hasDeno:    typeof Deno    !== 'undefined',
  };

  let runtime = 'unknown';
  let version = null;

  if (flags.hasDeno) {
    runtime = 'deno';
    version = Deno.version.deno;
  } else if (flags.hasProcess) {
    runtime = 'node';
    version = process.versions.node;
  } else if (flags.hasCaches && !flags.hasWindow) {
    runtime = 'cloudflare-workers';
  } else if (flags.hasWindow) {
    runtime = 'browser';
    version = typeof navigator !== 'undefined' ? navigator.userAgent.slice(0, 60) : null;
  }

  return { runtime, version, flags };
};

// 直接実行した場合のみ出力
if (typeof process !== 'undefined' && process.argv[1]?.endsWith('env-probe.mjs')) {
  const result = probe();
  console.log(JSON.stringify(result, null, 2));
  console.log(`\n▶ 実行環境: ${result.runtime}  ${result.version ?? 'version N/A'}`);
}
```

Node.jsでの実行は`node env-probe.mjs`で完結する。ブラウザはDevToolsコンソールに貼り付け、Cloudflare Workersは`wrangler dev`起動後のログで確認する。

## Claude API初期化前に`env-probe.mjs`を挟む統合パターン

Claude APIクライアントを初期化する前に診断を通すことで、「`process.env.ANTHROPIC_API_KEY`がundefined」「`fetch`が存在しない」といったエラーの原因を初期化ログの段階で特定できる。

```javascript
// claude-client.js
import Anthropic from '@anthropic-ai/sdk';
import { probe } from './env-probe.mjs';

const env = probe();

if (env.runtime === 'unknown') {
  throw new Error(`[env-probe] 実行環境を特定できません: ${JSON.stringify(env.flags)}`);
}
if (env.runtime === 'cloudflare-workers' && env.flags.hasProcess) {
  console.warn('[env-probe] Workers環境でprocess.envを参照中。wrangler.toml の [vars] を確認。');
}

const client = new Anthropic({
  apiKey: env.runtime === 'cloudflare-workers'
    ? globalThis.ANTHROPIC_API_KEY   // Workers binding から取得
    : process.env.ANTHROPIC_API_KEY,
});
```

スタックトレースに`[env-probe]`プレフィックスが乗るため、どの環境判定で落ちたかを1行で追える。

## 4環境の実行結果比較 — 出力パターンで症例を30秒で特定する

`runtime: 'unknown'`が出た場合、esbuild/Viteのバンドラーが環境変数を静的置換した結果、フラグが全て`false`になっている可能性がある。この症例は**第3章「esbuild `define` が`process.env`を消す地雷」**で扱う。

```
# Node.js v22.3.0
▶ 実行環境: node  22.3.0
flags: { hasWindow: false, hasProcess: true, hasCaches: false, hasDeno: false }

# ブラウザ (Chrome 125)
▶ 実行環境: browser  Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)...
flags: { hasWindow: true, hasProcess: false, hasCaches: true, hasDeno: false }

# Cloudflare Workers (wrangler dev)
▶ 実行環境: cloudflare-workers  version N/A
flags: { hasWindow: false, hasProcess: false, hasCaches: true, hasDeno: false }

# Deno 1.43.6
▶ 実行環境: deno  1.43.6
flags: { hasWindow: false, hasProcess: false, hasCaches: false, hasDeno: true }
```

## 第2〜5章への症例マップ — `env-probe`の出力をキーに引くデバッグ辞書

本書の残り4章は`env-probe.mjs`の出力をキーに引く辞書として設計した。

| `runtime` 出力 | 典型エラー文字列 | 該当章 |
|---|---|---|
| `node` | `ReferenceError: fetch is not defined` | 第2章 |
| `browser` | `Uncaught ReferenceError: process is not defined` | 第3章 |
| `cloudflare-workers` | `TypeError: Cannot read properties of undefined (reading 'get')` | 第4章 |
| `deno` | `error: Relative import path not prefixed with` | 第5章 |
| `unknown` | env-probe自体が動かない（バンドラー副作用） | 第5章補論 |

```bash
# 手元で今すぐ実行して症例番号を特定する
node env-probe.mjs | grep '"runtime"'
# => "runtime": "node" なら第2章へ、"browser" なら第3章へ直行
```

第2章からはエラー文字列を章タイトルにした辞書構成に切り替わる。`env-probe.mjs`で自分の症例番号を確定させてから購入すれば、当日中にデバッグを完結できる。
```
