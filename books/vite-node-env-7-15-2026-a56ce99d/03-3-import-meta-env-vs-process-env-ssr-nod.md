---
title: "第3章 import.meta.env vs process.env：SSR・Node APIサーバーでの両立配線"
free: false
---

本章のゴール：`import.meta.env`（Viteクライアント）と`process.env`（Node側dotenv）を同一リポジトリで衝突させずに配線し、欠落を起動0.1秒で落とすzodバリデーションを動かす。

## import.meta.env と process.env を取り違える3つの症状と原因

`import.meta.env.API_URL` が `undefined`、あるいは Express 側で `process.env.DATABASE_URL` が空になる事故は、参照する側を取り違えているのが原因の8割。下表で逆引きする。

| 症状 | 参照箇所 | 正しい側 |
|---|---|---|
| ブラウザで `undefined` | Vite client | `import.meta.env.VITE_*` |
| API サーバーで空 | Express/Fastify | `process.env.*` + dotenv |
| SSR 中だけ空 | render関数 | `import.meta.env`（VITE_接頭辞必須） |

```ts
// src/client/config.ts — クライアントは VITE_ 接頭辞のみ露出
export const API_URL = import.meta.env.VITE_API_URL  // ✅
// server/db.ts — Node は process.env、接頭辞不要
export const DSN = process.env.DATABASE_URL          // ✅
```

## Express APIサーバーで dotenv.config() の読込位置を間違えて undefined になる罠

`process.env.X` が `undefined` になる最頻原因は、`dotenv.config()` を「import より後」に呼ぶこと。ESMは`import`を巻き上げて先に評価するため、`db.ts`の`process.env`読込が先行して空になる。

```ts
// ❌ NG: import 評価時点で .env 未ロード → undefined
import { DSN } from './db'   // ここで process.env.DATABASE_URL を読む
import 'dotenv/config'        // 手遅れ

// ✅ OK: package.json で実行前に注入する
// "scripts": { "start": "node --env-file=.env dist/server.js" }
// Node 20.6+ なら --env-file が標準。dotenv 依存ごと消せる
```

## モノレポで envDir がずれて .env を読まない罠をディレクトリ構成で潰す

pnpm workspace で `apps/web/vite.config.ts` を動かすと、Viteはデフォルトで`apps/web/`配下の`.env`を探す。ルートに`.env`を置くと読まれない。`envDir`をルートに固定する。

```
repo/
├─ .env                  ← 共有変数（VITE_API_URL など）
├─ apps/web/vite.config.ts
└─ apps/api/server.ts
```

```ts
// apps/web/vite.config.ts
import { defineConfig } from 'vite'
import { resolve } from 'node:path'
export default defineConfig({
  envDir: resolve(__dirname, '../../'),  // ルートの .env を読む
  envPrefix: 'VITE_',
})
```

## zod でクライアント・サーバー両方を起動時に fail-fast 検証する

欠落を本番で気づくと痛い。zodで両環境を起動時に検証し、欠けていれば即座にプロセスを止める。CIでビルド前に1回走らせれば、デプロイ後のundefined事故をゼロにできる。

```ts
// shared/env.ts
import { z } from 'zod'

const serverSchema = z.object({
  DATABASE_URL: z.string().url(),
  PORT: z.coerce.number().default(3000),
})
const clientSchema = z.object({
  VITE_API_URL: z.string().url(),
})

export const serverEnv = serverSchema.parse(process.env)            // Node 側
export const clientEnv = clientSchema.parse(import.meta.env)         // Vite 側
// 欠落時: ZodError を投げて即終了 → 起動約0.1秒で fail-fast
```

検証は両入口（`server.ts`の先頭と`main.ts`の先頭）で1回 import するだけ。`parse`が落とすので、`undefined`がアプリ深部まで伝播する前に握り潰せる。
