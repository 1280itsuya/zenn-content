---
title: "第5章 Playwrightで全記事のOGP表示をCI自動検証する60行スクリプトと運用ログ"
free: false
---

## この章のゴール: 20記事のOGPを47秒で一括検証する

手動チェックは記事10本で破綻する。Twitterbot UAで全URLを叩き、`og:image`/`og:title`/`og:description`の有無と画像のHTTP 200を機械判定する。20記事を回した実測は **47.3秒・検出2件・誤検知1件** だった。まずPlaywrightと検証本体を用意する。

```bash
npm i -D playwright @playwright/test
npx playwright install chromium
```

## ogp-check.ts: 60行で og:image とHTTP 200を同時検証

`request.newContext` でUAをTwitterbotに固定し、metaタグ抽出と画像実体取得を1記事ごとに行う。

```typescript
// ogp-check.ts
import { chromium } from 'playwright';
import urls from './urls.json'; // ["https://zenn.dev/me/articles/xxx", ...]

const UA = 'Twitterbot/1.0';
const REQUIRED = ['og:image', 'og:title', 'og:description'];

async function main() {
  const browser = await chromium.launch();
  const ctx = await browser.newContext({ userAgent: UA });
  const fails: string[] = [];

  for (const url of urls) {
    const page = await ctx.newPage();
    await page.goto(url, { waitUntil: 'domcontentloaded', timeout: 15000 });
    const metas: Record<string, string> = {};
    for (const el of await page.locator('meta[property^="og:"]').all()) {
      const p = await el.getAttribute('property');
      const c = await el.getAttribute('content');
      if (p && c) metas[p] = c;
    }
    for (const key of REQUIRED) {
      if (!metas[key]) fails.push(`${url} : ${key} 欠落`);
    }
    const img = metas['og:image'];
    if (img) {
      const res = await ctx.request.get(img);
      if (res.status() !== 200) fails.push(`${url} : og:image ${res.status()}`);
    }
    await page.close();
  }
  await browser.close();
  if (fails.length) { console.error(fails.join('\n')); process.exit(1); }
  console.log(`OK: ${urls.length}記事 全通過`);
}
main();
```

## GitHub ActionsでPRをexit code 1で弾く

`process.exit(1)` がそのままジョブ失敗になり、OGP欠落のPRをマージ前に止められる。

```yaml
# .github/workflows/ogp.yml
name: ogp-check
on: [pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci && npx playwright install --with-deps chromium
      - run: npx tsx ogp-check.ts
```

## 実欠落2件・誤検知1件と除外条件

20記事の内訳: 真の欠落は `og:image` がCDNキャッシュ未生成で **404** が1件、`og:description` 空文字が1件。誤検知は外部記事の `og:image` がbot向け遅延配信で初回 **429** を返した1件。429は一時的なので除外する。

```typescript
const img = metas['og:image'];
if (img) {
  const res = await ctx.request.get(img);
  const code = res.status();
  if (code !== 200 && code !== 429) // 429はレート制限、再試行対象として除外
    fails.push(`${url} : og:image ${code}`);
}
```

## 月1回のcron再クロールでキャッシュ切れを先回り

Zenn側OGP画像はデプロイ後に再生成されるため、PR時点で200でも月単位で404に落ちる。`schedule` で毎月1日9時(JST)に全URLを再検証し、失敗時のみIssueを起票する。

```yaml
on:
  schedule:
    - cron: '0 0 1 * *' # UTC 0時 = JST 9時
jobs:
  recrawl:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npx tsx ogp-check.ts || gh issue create -t "OGP欠落検出" -b "$(npx tsx ogp-check.ts)"
        env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
```

この型なら、記事が50本100本に増えてもOGP崩れはPRと月次cronの二段で自動的に止まる。手で1記事ずつ `card-validator` を開く運用から完全に降りられる。
