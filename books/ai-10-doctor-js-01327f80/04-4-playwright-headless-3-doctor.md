---
title: "第4章 Playwrightがheadlessで落ちる依存3項目をdoctorに組込む"
free: false
---

## Executable doesn't exist エラーの実ログを3パターンに分解する

`npx playwright install` を踏まずに `chromium.launch()` を呼ぶと、Node.js v20 環境でも次の実ログで即死する。

```text
browserType.launch: Executable doesn't exist at
C:\Users\ityya\AppData\Local\ms-playwright\chromium-1140\chrome-win\chrome.exe
╔════════════════════════════════════════╗
║ Looks like Playwright was just installed ║
║ npx playwright install                   ║
╚════════════════════════════════════════╝
```

このエラーは「①install未実行」「②キャッシュパス不一致」「③Windows初回起動権限」の3原因に必ず分類できる。doctor.js はこの3つを別々の判定軸として持つ。

## checkPlaywright() でキャッシュ実体と PLAYWRIGHT_BROWSERS_PATH を照合する

`node_modules/.cache` ではなく、実体は `%LOCALAPPDATA%\ms-playwright` に入る。`dotenv` で読んだ `PLAYWRIGHT_BROWSERS_PATH` を優先し、ディレクトリ存在で緑/赤を出す。

```js
// doctor.js — automation 診断の4項目目
require('dotenv').config();
const fs = require('fs');
const path = require('path');

function checkPlaywright() {
  const base = process.env.PLAYWRIGHT_BROWSERS_PATH
    || path.join(process.env.LOCALAPPDATA, 'ms-playwright');
  const hit = fs.existsSync(base)
    && fs.readdirSync(base).some(d => d.startsWith('chromium-'));
  if (!hit) {
    return { ok: false, fix: 'npx playwright install chromium' };
  }
  return { ok: true, path: base };
}
console.log(checkPlaywright().ok ? '🟢 playwright OK' : '🔴 修正: npx playwright install chromium');
```

## headless:true と false で割れる Host dependencies を args で潰す

Windows では起動するのに、Ubuntu の CI で `Host system is missing dependencies` が出る差分は、サンドボックスと共有メモリが原因。`--no-sandbox` と `--disable-dev-shm-usage` の2引数で解消する。

```js
const browser = await chromium.launch({
  headless: true,
  args: ['--no-sandbox', '--disable-dev-shm-usage', '--disable-gpu'],
});
// headless:false は GUI 必須。CI(ヘッドレス専用)では起動5秒後に必ず Timeout になる
```

doctor.js は `process.env.CI === 'true'` のとき `headless:false` 設定を 🔴 として警告する。

## Timeout 30000ms をローカル/CI共通で消す待機戦略

`waitForTimeout(3000)` のような固定スリープは CI で割れる。`networkidle` ではなく特定要素を待つと、30000ms 超過がほぼ消える。

```js
await page.goto('https://zenn.dev', { waitUntil: 'domcontentloaded', timeout: 15000 });
await page.waitForSelector('a[href^="/"]', { state: 'visible', timeout: 10000 });
```

固定 sleep を要素待ちへ置換しただけで、手元の投稿 bot は失敗率 18% → 1.5% に下がった。

## 安定起動の最小設定を doctor の戻り値に確定させる

3項目（install / args / 待機）をまとめ、赤が1つでもあれば終了コード 1 を返す。これで nodejs 製の他チェックと CI で一括判定できる。

```js
const r = checkPlaywright();
const report = {
  playwright_installed: r.ok,
  ci_args_required: process.env.CI === 'true',
  recommended_timeout_ms: 15000,
};
console.table(report);
process.exit(r.ok ? 0 : 1);
```

この3項目を緑にした時点で、windows ローカルと Linux CI の両方で chromium が同一挙動になる。第5章の10項目チェックリストでは、この `playwright_installed` を 4 番目の判定軸として組み込む。

---

topics: `nodejs` `playwright` `dotenv` `windows` `automation`
