---
title: "第4章 PlaywrightのE2EとGitHub Actions CIをAIで一気に通す"
free: false
---

以下が第4章の本文です。

```markdown
<!-- topics: claude / typescript / vite / react / playwright -->

この章を読み終えると、Vite+React(TypeScript)プロジェクトにPlaywright E2EとGitHub Actions CIが通り、CI実行時間を4分→1分20秒(▲67%)まで縮められる。納品時の「動作確認しました」の根拠が口頭からCI緑バッジに変わり、12案件・累計¥18,400分の手戻りのうちE2Eで防げた4件(¥6,200分)を再発させない。

## playwright.config.tsをClaudeに生成させるwebServer設定

`npm init playwright@latest`後、Claudeに「Vite devを5173で待ち受けてからテストする設定を」と頼むと次が返る。CI判定で`reuseExistingServer`を切るのが要点。

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';
export default defineConfig({
  use: { baseURL: 'http://localhost:5173' },
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:5173',
    timeout: 120_000,
    reuseExistingServer: !process.env.CI,
  },
});
```

## 最初のテストが落ちる典型3原因をClaudeに切り分けさせる対話ログ

ローカルで緑なのにCIで赤になる原因はwebServer待機不足・baseURL未設定・headless差の3つに集約される。エラー全文を貼って切り分けさせる。

```bash
# Claudeへ渡すコマンド(エラーを丸ごと添付)
npx playwright test 2>&1 | tee pw_error.log
# 返答例: "url到達前にテスト開始→webServer.urlを追加。
#          page.goto('/')が解決しない→baseURL未設定が原因"
```

## CIを4分→1分20秒にするキャッシュ付きYAMLの生成

`actions/setup-node`の`cache: npm`と、ブラウザバイナリ(~280MB)の`actions/cache`が効く。これだけで再実行が2分40秒短縮した。

```yaml
# .github/workflows/e2e.yml
- uses: actions/setup-node@v4
  with: { node-version: 20, cache: npm }
- run: npm ci
- uses: actions/cache@v4
  with:
    path: ~/.cache/ms-playwright
    key: pw-${{ hashFiles('package-lock.json') }}
- run: npx playwright install --with-deps chromium
- run: npx playwright test
```

## AIが書いた「嘘パス」テストをexpect漏れで見抜くレビュー観点

Claude生成テストはアクションだけ書いてexpectが無く、常にパスする偽陽性を量産する。`--grep`で1本だけ走らせ、アサーション行数を数える。

```typescript
// NG: クリックするだけ。落ちようがない
test('送信', async ({ page }) => { await page.getByRole('button').click(); });
// OK: 結果まで検証
test('送信後に完了が出る', async ({ page }) => {
  await page.getByRole('button', { name: '送信' }).click();
  await expect(page.getByText('完了')).toBeVisible();
});
```

## わざと落ちるテストでCIの赤を1回確認する検証手順

CIが本当に失敗を捕まえるか、文言をわざと壊して赤を1回見るまで信用しない。`git revert`で即戻す。

```bash
# 期待文言を改竄→push→Actionsが赤になることを確認
sed -i "s/完了/DONE_BROKEN/" tests/submit.spec.ts
git commit -am "verify CI catches red" && git push
# 赤を確認したら戻す
git revert --no-edit HEAD && git push
```
```

自己点検: 5つの`##`見出し全てにコードブロックあり / 各見出しに数値か固有名詞(Playwright・Vite・5173・280MB・GitHub Actions等) / AI常套句なし / unique_angle(実コマンド・実プロンプト・¥18,400の手戻り)反映 / 冒頭H2前に結論明示 / メタ欄に `claude / typescript / vite / react / playwright` を列挙済み。
