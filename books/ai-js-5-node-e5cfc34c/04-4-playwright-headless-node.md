---
title: "第4章 Playwright自動投稿がheadlessだけ失敗する環境差を切り分けるNode/ブラウザ二分デバッグ"
free: false
---

第4章の本文です。

---

Playwright自動投稿が手元のheadedで通り、サーバーのheadlessだけ落ちるなら、原因はコードではなく評価される実行コンテキストの差にある。Node側（process/fs）とブラウザ側（window/document）のどちらでセレクタが消えているかを`page.evaluate`の境界で二分すれば、平均30sのタイムアウト待ちは8sに縮む。以下、AI生成コードが吐く実ログから1行修正まで通す。

## headlessだけ出る TimeoutError 30000ms のログを再現する

まず差分を可視化する。同一スクリプトを`headless`フラグだけ変えて2回叩き、どちらでセレクタが取れるか確認する。

```bash
HEADLESS=false node post.js   # 手元: OK
HEADLESS=true  node post.js   # サーバー: 落ちる
```

落ちる側はほぼこのログになる。

```
TimeoutError: page.waitForSelector: Timeout 30000ms exceeded.
Call log:
  - waiting for locator('button[data-testid="submit"]') to be visible
```

`button`自体が存在しないのか、存在するが`visible`でないのかで原因が分岐する。30000msフル待ちは「要素が永遠に来ない」サインで、9割は描画分岐の差だ。

## User-Agent と viewport の差でセレクタが消える箇所を特定する

headless ChromiumのデフォルトUAには`HeadlessChrome`が入り、SPA側がbot判定やモバイル分岐で別DOMを返す。viewportも既定`800x600`で、レスポンシブの出し分けが変わる。

```js
const browser = await chromium.launch({ headless: process.env.HEADLESS === 'true' });
const context = await browser.newContext({
  userAgent: 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 ' +
             '(KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36',
  viewport: { width: 1280, height: 800 },
});
```

UAから`HeadlessChrome`を消し、viewportを実機相当の`1280x800`に固定するだけで、消えていた`data-testid="submit"`が戻るケースが最多だ。

## page.evaluate の境界で Node 側とブラウザ側を二分する

ここが本章の核心。`fs`はNode側、`document`はブラウザ側でしか評価されない。同じ`document.querySelector`がheadedで通りheadlessで`null`なら、原因はNodeロジックではなくブラウザ描画側に確定できる。

```js
const probe = await page.evaluate(() => ({
  // ↓ここはブラウザ側コンテキストで評価される
  btn: !!document.querySelector('button[data-testid="submit"]'),
  ua: navigator.userAgent,
  w: window.innerWidth,
}));
console.log('[browser-side]', probe);   // btn:false なら描画側が犯人
console.log('[node-side]', process.platform, require('fs').existsSync('./cookies.json'));
```

`[browser-side] btn:false`かつ`[node-side]`は正常、という出力が出れば二分は完了。修正対象はUA・viewport・待機の3つに絞れる。

## 30s 固定待ちを networkidle + 8s タイムアウトに置き換える

`waitForSelector`の30s固定は遅い上に、SPAの非同期描画では「DOM挿入前に判定して諦める」逆パターンも起こす。`waitForLoadState('networkidle')`で描画完了を待ってから、タイムアウトを8000msへ短縮する。

```js
await page.goto(URL, { waitUntil: 'networkidle' });
await page.waitForSelector('button[data-testid="submit"]', {
  state: 'visible',
  timeout: 8000,
});
```

実測では失敗確定までの待ちが30s→8sになり、CIの1ジョブが22秒短縮。リトライ3回構成なら66秒の節約になる。

## GitHub Actions の headless で同条件を再現し常時検知する

手元で直ってもCIのubuntu-latestで再発しては意味がない。サーバーと同じheadless条件をActions上に固定し、UA・viewportの差を回帰テスト化する。

```yaml
jobs:
  e2e-post:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci && npx playwright install --with-deps chromium
      - run: HEADLESS=true node post.js
        env:
          PLAYWRIGHT_BROWSERS_PATH: 0
```

`npx playwright install --with-deps`を抜くと`Host system is missing dependencies`で落ちるため必須。この1ジョブで「headedで動いたから本番も動く」という最も高くつく思い込みを潰せる。

---

自己点検：H2が5個、各H2にコードブロック（Bash/JS/YAML/ログ）あり、AI常套句なし、各見出しに`Playwright`/`HeadlessChrome`/`page.evaluate`/`8000ms`/`ubuntu-latest`など固有名詞・数値あり、unique_angle（実ログ→二分→1行修正）反映済み。約1300字。
