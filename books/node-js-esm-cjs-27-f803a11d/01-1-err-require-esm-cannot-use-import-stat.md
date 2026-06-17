---
title: "第1章：ERR_REQUIRE_ESM・Cannot use import statementの正体と3分診断フロー（無料）"
free: true
---

章の目的を確認し、本文を執筆します。

---

## ERR_REQUIRE_ESM と Cannot use import statement：2大エラーの正体

この2つは同じ根本原因から生まれる**別物**だ。混同すると修正が逆方向になる。

| エラー | 発火条件 |
|--------|---------|
| `ERR_REQUIRE_ESM` | CJS環境で `require()` → ESMパッケージを読もうとした |
| `SyntaxError: Cannot use import statement in a module` | CJS扱いのファイルに `import` 文を書いた |

Node.js 18以降、**OpenAI SDK 4.x / Anthropic SDK 0.24以降**はパッケージ自体が `"type": "module"` で配布されている。古いCJSプロジェクトで `require('openai')` と書いた瞬間、下記が爆発する。

```bash
node -e "require('openai')"
# Error [ERR_REQUIRE_ESM]: require() of ES Module
# node_modules/openai/index.js not supported.
# Instead change the require() to a dynamic import().
```

`Cannot use import statement` は逆向きの衝突だ。Node.jsがCJSと認識しているファイルに `import openai from 'openai'` と書けば発火する。発火条件が異なるため、修正コードも分岐する。

## Node.js がCJS/ESMを決定する3つのルール（優先順）

Node.jsは以下の順序で実行コンテキストを確定する。この優先順を知らないと診断が迷走する。

1. 拡張子が `.mjs` → **ESM**（無条件）
2. 拡張子が `.cjs` → **CJS**（無条件）
3. `.js` の場合 → 最近傍 `package.json` の `"type"` フィールドで決まる

```json
// package.json
{
  "name": "my-app",
  "type": "module"
  // "type": "commonjs" または省略 → .js全部CJS
}
```

`tsconfig.json` の `"module": "CommonJS"` は**TypeScriptのコンパイル出力**を制御するものであり、Node.jsの実行時判定とは別軸だ。tscがCJS `.js` を出力しても、`package.json` に `"type": "module"` があればNode.jsはESMとして実行し衝突する。この二重管理が混乱の最大の温床になる。

## 3点確認チェックリスト：package.json → import/require → tsconfig.module

診断に必要な情報を3コマンドで揃える。

```bash
# ① 自プロジェクトの type を確認
node -e "const p=require('./package.json'); console.log('type:', p.type ?? '(省略=CJS)')"

# ② 使用中のAI SDKがESMかCJSか確認
node -e "const p=require('./node_modules/openai/package.json'); console.log('openai type:', p.type)"
node -e "const p=require('./node_modules/@anthropic-ai/sdk/package.json'); console.log('anthropic type:', p.type)"

# ③ tsconfig.json のmodule設定確認（TypeScriptプロジェクトのみ）
node -e "const t=require('./tsconfig.json'); console.log('tsconfig module:', t.compilerOptions?.module)"
```

`openai type: module` かつ 自プロジェクト `type: (省略=CJS)` の組み合わせが最も頻出のERR_REQUIRE_ESM発火パターンだ。

## 5秒判定フローチャート：CJS / ESM / ハイブリッドの3択

上記3コマンドの出力を以下のフローチャートに当てはめる。

```
package.json に "type": "module" がある？
├─ YES → ESMプロジェクト
│        require() をすべて import / dynamic import() に変換 → 第3章
└─ NO  → CJSプロジェクト（省略=CJS）
         ↓
         openai / @anthropic-ai/sdk を使っている？
         ├─ YES → ERR_REQUIRE_ESM 発火構成（要対処）
         │        A: "type":"module" 追加でESM全移行   → 第4章
         │        B: dynamic import() で局所回避       → 第5章
         │        C: CJS互換バージョンにピン留め        → 第6章
         └─ NO  → 現状は安全。SDK追加前に第2章を読む

拡張子 .mjs/.cjs 混在？ → ハイブリッド構成（第7章）
```

自プロジェクトがどの象限に属するかが確定すれば、次章以降の修正コードを迷わず引ける。

## 第2章以降で解決する27パターンの射程

本章で「なぜ起きるか」の全体像は揃った。しかし実案件では診断より**修正で詰まる**ことが多い。第2章以降では実エラーメッセージを入口にした27パターンを収録している。

```
#8  Jest / Vitest でAI SDKをモック → Must use import to load ES Module
#11 Next.js 14 app/ 配下で require() → Cannot require ESM
#15 tsx コマンドで実行中なのに ERR_REQUIRE_ESM が消えない
#22 Electron mainプロセスでOpenAI SDKが読み込めない
#27 monorepo内パッケージ間のtype不一致で片方だけ爆発する
```

各パターンは「再現コード → エラー全文 → 修正コード → 所要時間の目安」で統一している。次章「Jest×Anthropic SDK」のモック設定は、AI SDK導入直後に8割のプロジェクトで踏む罠なので、そこから読み始めることを推奨する。
