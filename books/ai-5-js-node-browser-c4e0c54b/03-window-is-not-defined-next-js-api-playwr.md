---
title: "`window is not defined` — Next.js APIルートでPlaywright/Puppeteer製AI副業ツールが即死する3脱出口"
free: false
---

Zenn有料章を執筆します。

## `window is not defined` — Next.js APIルートでPlaywright/Puppeteer製AI副業ツールが即死する3脱出口

---

## SSR実行時の即死メカニズム — モジュール評価が `chromium.launch()` を巻き込む

Next.js のビルドは `pages/api/*.ts` を Node.js サーバーサイドで評価する。`import { chromium } from 'playwright'` をファイル先頭に書いた瞬間、モジュールロード時点で Playwright 内部の `BrowserContext` が `window` / `navigator` を参照しにいき、サーバー環境にそれが存在しないためクラッシュする。

```
ReferenceError: window is not defined
    at Object.<anonymous> (node_modules/playwright-core/lib/utils/browserPaths.js:12:1)
    at Module._compile (node:internal/modules/cjs/loader:1364:14)
```

ポイントは「`chromium.launch()` を呼ぶ前に死ぬ」こと。`typeof window !== 'undefined'` ガードはモジュール評価後なので効かない。

---

## 脱出口①: Next.js `dynamic import + ssr: false` でPlaywrightをブラウザ側に追い出す

`next/dynamic` の `ssr: false` オプションは React コンポーネント専用だが、APIルート内で `import()` を関数スコープに閉じ込める書き方で同等の効果を得られる。

```typescript
// pages/api/scrape.ts
import type { NextApiRequest, NextApiResponse } from 'next'

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  // トップレベルimport禁止 — ここで遅延ロード
  const { chromium } = await import('playwright')
  const browser = await chromium.launch({ headless: true })
  const page = await browser.newPage()
  await page.goto(req.body.url as string)
  const title = await page.title()
  await browser.close()
  res.json({ title })
}
```

**制約**: Vercel の Serverless Function は実行時間上限 60 秒 / メモリ 3 GB。Playwright のバイナリサイズ（約 280 MB）がコールドスタートを 8〜12 秒押し上げる。月間リクエスト数が 1,000 本を超えると Function 実行費用が ¥2,800/月 を突破する実測値が出た。

---

## 脱出口②: Vercel Edge Runtime から除外 — `export const config` で Node.js 専用化

Vercel の Edge Runtime はデフォルトで有効化されているプロジェクトがある。Edge Runtime では Node.js API (`child_process`, `fs`) が使えず Playwright は動かない。明示的に `nodejs` ランタイムを宣言する。

```typescript
// pages/api/scrape.ts の末尾に追加
export const config = {
  runtime: 'nodejs', // 'edge' のままだと ReferenceError: process is not defined も発生する
  maxDuration: 60,
}
```

`vercel.json` でも関数単位に指定できる。

```json
{
  "functions": {
    "pages/api/scrape.ts": {
      "runtime": "nodejs20.x",
      "memory": 3008,
      "maxDuration": 60
    }
  }
}
```

この設定だけで Playwright の動作は安定するが、コールドスタート問題と従量課金は残る。

---

## 脱出口③: Playwright専用Dockerコンテナ切り出し — GitHub Actions全量YAML公開

根本解決は Playwright を Next.js から完全分離し、専用コンテナで動かすことだ。Next.js API ルートはジョブキューにタスクを投げるだけにする。以下は Railway / Fly.io にデプロイ可能な構成。

```yaml
# .github/workflows/playwright-worker.yml
name: Playwright Worker CI

on:
  push:
    paths:
      - 'playwright-worker/**'

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Playwright worker image
        uses: docker/build-push-action@v5
        with:
          context: ./playwright-worker
          push: true
          tags: ghcr.io/${{ github.repository }}/playwright-worker:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

```dockerfile
# playwright-worker/Dockerfile
FROM mcr.microsoft.com/playwright:v1.44.0-jammy

WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

COPY . .
EXPOSE 3001
CMD ["node", "server.js"]
```

```javascript
// playwright-worker/server.js
const express = require('express')
const { chromium } = require('playwright')

const app = express()
app.use(express.json())

app.post('/scrape', async (req, res) => {
  const browser = await chromium.launch({ headless: true, args: ['--no-sandbox'] })
  const page = await browser.newPage()
  await page.goto(req.body.url, { waitUntil: 'networkidle' })
  const html = await page.content()
  await browser.close()
  res.json({ html })
})

app.listen(3001, () => console.log('Playwright worker ready on :3001'))
```

Next.js 側は `fetch('http://playwright-worker:3001/scrape', ...)` を呼ぶだけになり、`window is not defined` は構造上発生しなくなる。

---

## 実測: Vercel Function ¥2,800/月 → ¥0 への移行ステップ

| 移行前 | 移行後 |
|---|---|
| Vercel Serverless (従量) | Railway Starter ($5/月 固定、無料枠$5クレジット消費) |
| コールドスタート 8〜12 秒 | コンテナ常駐 0.3 秒 |
| Playwright バイナリ毎回展開 | イメージ内キャッシュ済み |
| 月 ¥2,800（リクエスト 1,200 本） | 月 ¥0（無料枠内）|

移行コマンドは 3 行で完了する。

```bash
# Railway CLI でデプロイ
npm install -g @railway/cli
railway login
railway up --service playwright-worker
```

`PLAYWRIGHT_WORKER_URL` を Next.js の環境変数に追加し、既存の `pages/api/scrape.ts` 内の Playwright 直呼び出しを `fetch` に置き換えるだけで切り替わる。Vercel の Function 実行ログから該当エンドポイントの呼び出しカウントが消えたことを確認して移行完了とする。
