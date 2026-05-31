---
title: "第4章 12回失敗ログ: 依存衝突とESM/CJS地獄をClaudeで抜けた回避策"
free: false
---

## 事故1〜3: peerDeps衝突で `npm install` が3回中3回コケた損失38分

最初の事故は React 19 と `@testing-library/react@14` の peerDeps 衝突。`npm install` が `ERESOLVE` で停止し、`--legacy-peer-deps` で逃げた結果、後でVitestが動かなくなった。Claudeへ投げた質問文がこれ:

```bash
# Claudeに渡した実エラー（全文貼り付けが鍵）
npm error code ERESOLVE
npm error While resolving: @testing-library/react@14.3.1
npm error Found: react@19.0.0
# 質問: 「このERESOLVEを--forceで握りつぶさず、正しいバージョンに揃える差分だけ出して」
```

返ってきた解決差分は `@testing-library/react@16.1.0` への更新1行。`--legacy-peer-deps` を消せて、損失38分のうち再発分はゼロになった。

## 事故4〜6: 「Cannot use import statement outside a module」をESM固定で潰す

CJS前提のスクリプトに `import` を書いて即死。`package.json` の `type` 未指定が原因で、Claudeに「`type: module` 固定で全スクリプトをESMに寄せる最小差分」を要求した。

```json
{
  "type": "module",
  "scripts": {
    "dev": "vite",
    "test": "vitest run"
  }
}
```

CJS依存が残る場合は拡張子で分離する。`*.cjs` に逃がす1手で混在事故3件が止まった。

```js
// scripts/legacy-report.cjs ← .cjs拡張子でCJSを明示
const fs = require("fs");
fs.writeFileSync("report.json", JSON.stringify({ ok: true }));
```

## 事故7〜9: Vitest + TS path エイリアス未解決を `vite-tsconfig-paths` で15分短縮

`@/utils` が `tsconfig.json` では通るのにVitestで `Cannot find module`。原因はVitestが `paths` を読まないこと。検索する前にチェックリスト第41項目へ飛べるよう、解決を1ファイルに固定した。

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";
import tsconfigPaths from "vite-tsconfig-paths";

export default defineConfig({
  plugins: [tsconfigPaths()],          // tsconfigのpathsをVitestへ伝播
  test: { environment: "node", globals: true },
});
```

これで `@/` エイリアスのテスト4本が一括で緑になり、手動 `alias` 記述（毎回12分）が不要になった。

## 事故10〜12を第39〜44項目へ: Claudeに事故ログ自体を読ませる回避手順

残り3件（Biome未設定でのlint暴走、`engines` 不一致、`.gitignore` 漏れ）はチェックリスト第39〜44項目に昇格させた。再発防止の核心は、感想ではなく事故ログをそのままClaudeへ食わせること。

```bash
# 12件の事故ログをまとめてClaudeに読ませる再利用プロンプト
cat docs/incident-log.md | \
  claude -p "この12件の事故を回避する初期構築を、package.json/vitest.config.ts/biome.json の差分で出力。ESM固定・peerDeps整合・path伝播を満たすこと"
```

12回の失敗で固めた47項目のうち、本章は第39〜44項目を埋める。次に同じスタックを組むときは `incident-log.md` を貼るだけで、90分が再び10分に戻る。

```yaml
# .github/workflows/verify.yml ← 事故再発を自動検知
on: [push]
jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci          # ERESOLVE再発をCIで即検出
      - run: npm test
```

---

topics: ["claude","typescript","vite","biome","githubactions"]
