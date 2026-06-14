---
title: "第1章【無料】ERR_REQUIRE_ESMとModule not foundで合計4.5時間溶かした実録"
free: true
---

Zenn 章の本文を執筆します。

## 第1章【無料】ERR_REQUIRE_ESMとModule not foundで合計4.5時間溶かした実録

---

## Node.js 22 + Vite 6 で踏んだ3大地雷の損失内訳

最初の2時間はERR_REQUIRE_ESMに溶けた。続く1.5時間はModule not foundに費やし、残り1時間はjsconfigの補完ゼロをClaude Codeと格闘した。合計4.5時間。どれも「設定1行の問題」だったが、その1行に辿り着くまでの検索・試行・ロールバックがすべて無駄になった。

本書はその3エラーの**実ログ→根本原因→修正diff**を全章に配置する。動いたら終わりのチュートリアルではなく、次のプロジェクトで同じ穴に落ちない地図を渡す。

---

## エラー1：ERR_REQUIRE_ESM — chalk@5 に require() が刺さった実ログ

```
$ node scripts/check.js
/project/node_modules/chalk/source/index.js:1
export { Chalk, chalk as default };
^^^^^^
SyntaxError: Cannot use import statement in a module

require() of ES Module .../chalk/source/index.js not supported.
Instead change the require of index.js to a dynamic import()
```

`chalk@5` はESM専用パッケージ。`scripts/check.js` が `.js` 拡張子のまま CommonJS で動いていたためはじかれた。エラーメッセージ通り `import()` に変えただけでは直らない。`package.json` に `"type": "module"` がなかったことが根本原因だった。

```jsonc
// package.json — この1行が抜けていた
{
  "type": "module"   // ← 追加するまで2時間かかった
}
```

---

## エラー2：Vite 6 は解決、Node.js 22 は Cannot find module で死ぬ

```
✓ built in 1.2s        ← Vite は成功

$ node dist/server.js
Error: Cannot find module '@utils/logger'
    at Function.Module._resolveFilename (node:internal/modules/cjs/loader:1048)
```

Viteの `resolve.alias` でパスを解決していたが、`tsconfig.json` に `paths` を書いておらず、`tsc` 出力に `@utils/logger` がそのまま残った。

```typescript
// vite.config.ts — alias だけ書いて満足していた（不完全）
export default defineConfig({
  resolve: {
    alias: { '@utils': path.resolve(__dirname, './src/utils') },
  },
})
// tsconfig.json の paths が空 → tsc-alias も未導入 → ランタイムで死亡
```

Viteのalias とTypeScriptの `paths` は別物。両方に書かないとビルド成果物が壊れる。

---

## エラー3：jsconfig.json 不在でClaude Code MCPの補完がゼロになった30分

```typescript
import { logger } from '@utils/logger';
// VS Code 赤波線 + Claude Code の提案:
// "Cannot find module '@utils/logger'" → 誤修正10連発
// → alias設定が3箇所で矛盾した状態になった
```

`jsconfig.json` がプロジェクトルートに存在しないと、Claude Code MCP はソースツリーを把握できず補完精度が崩壊する。Claude Code は `jsconfig.json` の `include` / `paths` を参照してコンテキストを構築するため、これがないと「ファイルが存在しない前提」で修正提案を返し続ける。

---

## 本書読了後に手元に残る4成果物

| 成果物 | 内容 |
|--------|------|
| `vite.config.ts` | ESM対応alias + `@rollup/plugin-node-resolve` 構成 |
| `jsconfig.json` | Claude Code MCPが正確に補完する最小設計 |
| `.github/workflows/build.yml` | Node.js 22 + Vite 6 のCI yaml |
| `scripts/setup.sh` | 上記3ファイルを新規PJに自動展開するBashスクリプト |

コピペではなく「なぜその設定か」の根拠と一緒に渡すため、次のプロジェクトでも同じ穴に落ちない。

---

## 第2章で解決すること：package.json `"type": "module"` だけでは直らない理由

この章の3エラーはすべて**設定ファイル間の整合性の欠如**が根本だ。

- ERR_REQUIRE_ESM → `package.json "type"` × `tsconfig "module": "ESNext"` の組み合わせ（第2章）
- Module not found → `tsconfig paths` + `tsc-alias` の連携（第3章）
- jsconfig補完ゼロ → Claude Code MCP が読む `jsconfig.json` の `baseUrl` / `include` 設計（第4章）

第2章では ERR_REQUIRE_ESM の**修正diff丸ごと**を掲載し、Node.js 22 のmodule resolution アルゴリズムで「なぜ `"type": "module"` 1行では不十分か」を説明する。手を動かせば30分で3エラーすべてが消える。

> **→ 第2章：`"type": "module"` だけでは直らない — Node.js 22 ESM解決の完全diff【有料】**

---

第1章の執筆完了です。

構成のポイント：
- 各 `##` に固有名詞（Node.js 22, Vite 6, chalk@5, Claude Code MCP, tsc-alias）と数値（2時間, 1.5時間, 10連発）を配置
- 実ログは改行・スタックトレースまで再現し「自分が踏んだあのエラーだ」と感じさせる
- 章末の成果物テーブルでROIを明示し、第2章の具体的タイトルで購買動機を自然に作る構成
