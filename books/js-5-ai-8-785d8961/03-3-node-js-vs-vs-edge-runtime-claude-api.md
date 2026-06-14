---
title: "第3章：Node.js vs ブラウザ vs Edge Runtime——Claude API呼び出しを3環境で動かす最小差分コード全公開"
free: false
---

## エラーログ実物：Edge RuntimeでAnthropicSDKを呼ぶと即死する3パターン

Vercel Edge Functionsに `@anthropic-ai/sdk` を素直にimportすると、デプロイ直後に以下のいずれかが出る。

```
# パターンA: Cloudflare Workers
Error: The 'net' module is not compatible with the CF Workers runtime.

# パターンB: Vercel Edge Runtime
DynamicServerError: Dynamic server usage: Route /api/chat requires ...
Module not found: Can't resolve 'tls'

# パターンC: Next.js App Router (middleware)
TypeError: Unable to determine runtime environment
  at AnthropicError (/node_modules/@anthropic-ai/sdk/src/error.ts:12)
```

3つとも**根本は同じ1行**が原因だ。

---

## 根本原因1行：`net`/`tls`/`http2`への暗黙依存を特定するdepcruise

```bash
npx depcruise --include-only "^@anthropic-ai/sdk" \
  --output-type text node_modules/@anthropic-ai/sdk/src
```

出力の中に `http2`, `net`, `tls` が現れる。AnthropicSDK v0.21以前はNode.js専用HTTPクライアントを内部で持っており、Edge Runtimeが提供しないビルトインモジュールを要求する。v0.24以降はfetch-firstに書き直されているが、**importパスを間違えると旧パスが混入する**。

---

## Edge-safe実装：fetch1本でClaude Haiku 3.5を叩く最小コード

```typescript
// app/api/edge-chat/route.ts
export const runtime = 'edge'; // Vercel Edge Runtime

export async function POST(req: Request) {
  const { prompt } = await req.json();

  const res = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'x-api-key': process.env.ANTHROPIC_API_KEY!,
      'anthropic-version': '2023-06-01',
      'content-type': 'application/json',
    },
    body: JSON.stringify({
      model: 'claude-haiku-4-5-20251001',
      max_tokens: 1024,
      messages: [{ role: 'user', content: prompt }],
    }),
  });

  const data = await res.json();
  return Response.json({ text: data.content[0].text });
}
```

SDKを一切使わない。`fetch`はEdge RuntimeとNode.js 18+の両方でネイティブに動く。polyfillは不要。

---

## Node.js専用環境でのStreamingとの差分：3行の書き換えで対応

Node.js（API Routes / Express）で逐次出力するならSDKのstream APIが使える。上記Edge実装との差分はこれだけだ。

```typescript
// Node.js専用：pages/api/stream-chat.ts
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic(); // ANTHROPIC_API_KEY を自動読取

export default async function handler(req, res) {
  res.setHeader('Content-Type', 'text/event-stream');

  const stream = await client.messages.stream({
    model: 'claude-haiku-4-5-20251001',
    max_tokens: 1024,
    messages: [{ role: 'user', content: req.body.prompt }],
  });

  for await (const chunk of stream) {
    if (chunk.type === 'content_block_delta') {
      res.write(`data: ${chunk.delta.text}\n\n`);
    }
  }
  res.end();
}
```

Edge版との違いは`client.messages.stream`の1行と`for await`ループのみ。**SDKを使う＝Node.js専用、fetchを使う＝Edge対応**、この対応を頭に入れれば迷わない。

---

## Playwright/Puppeteerがブラウザ環境で絶対に動かない理由と代替2択

Playwrightはブラウザを**外部プロセスとして起動**するため、`child_process`と`fs`が必須だ。ブラウザのJavaScript実行環境にはどちらも存在しない。

```
Error: Cannot find module 'child_process'
    at Function.Module._resolveFilename (node:internal/modules/cjs/loader)
```

これが出たら環境を変えるしかない。コード修正では解決しない。代替手段は2択だ。

```yaml
# 代替A: Playwright MCP（ローカルNode.jsに任せる）
# .mcp.json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    }
  }
}

# 代替B: Serverless Chrome（Vercel/Lambda上で動かす）
# package.json
{
  "dependencies": {
    "@sparticuz/chromium": "^123.0.0",
    "playwright-core": "^1.44.0"
  }
}
```

副業ツールのスクレイピング処理はAPI側（Node.js）に集約し、Edgeに置かないのが基本設計だ。

---

## 汎用環境判定ユーティリティ：どのコードベースにも貼り込める30行

```typescript
// lib/env-detect.ts

type RuntimeEnv = 'node' | 'edge' | 'browser' | 'unknown';

export function detectRuntime(): RuntimeEnv {
  if (typeof process !== 'undefined' && process.versions?.node) {
    return 'node';
  }
  if (typeof EdgeRuntime !== 'undefined') {
    return 'edge';
  }
  if (typeof window !== 'undefined') {
    return 'browser';
  }
  return 'unknown';
}

export function requiresNativeClient(): boolean {
  return detectRuntime() === 'node';
}

// 使用例
import Anthropic from '@anthropic-ai/sdk';

export async function callClaude(prompt: string): Promise<string> {
  const runtime = detectRuntime();

  if (runtime === 'node') {
    // SDKのstream APIが使える
    const client = new Anthropic();
    const msg = await client.messages.create({
      model: 'claude-haiku-4-5-20251001',
      max_tokens: 512,
      messages: [{ role: 'user', content: prompt }],
    });
    return (msg.content[0] as { text: string }).text;
  }

  // Edge / Worker: 素のfetchにフォールバック
  const res = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'x-api-key': process.env.ANTHROPIC_API_KEY!,
      'anthropic-version': '2023-06-01',
      'content-type': 'application/json',
    },
    body: JSON.stringify({
      model: 'claude-haiku-4-5-20251001',
      max_tokens: 512,
      messages: [{ role: 'user', content: prompt }],
    }),
  });
  const data = await res.json();
  return data.content[0].text;
}
```

この `callClaude` 関数をNext.js・Cloudflare Workers・Expressのいずれに貼っても動く。副業ツールの複数環境デプロイで環境判定を書き直す手間がゼロになる。
