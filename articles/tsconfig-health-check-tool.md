---
title: "tsconfigの設定ミスを貼るだけで診断するツールを作った（20項目チェック）"
emoji: "🩺"
type: "tech"
topics: ["typescript", "tsconfig", "フロントエンド", "個人開発"]
published: true
---

tsconfig.json、「どこかからコピペしたまま中身はよく分かっていない」という人がかなり多いと思います。私もそうでした。

動いているうちはいいのですが、`moduleResolution` と `module` の組み合わせが噛み合っていなかったり、`strict` が実は切れていたり、`paths` を書いたのにランタイム側の対応を忘れていたり——「あとで謎エラーとして爆発する地雷」が tsconfig には埋まりがちです。

そこで、**tsconfig.json を貼り付けるだけで設定ミスや古い設定を20項目チェックしてスコア化するツール**を作りました。

https://1280itsuya.github.io/devtools/tsconfig/

- 登録不要・無料
- **すべてブラウザ内で動作**（tsconfigの中身はどこにも送信されません）
- コメント付き（JSONC）のまま貼ってOK

## 何をチェックするのか

実際にハマった人が多い順に、代表的な診断項目を紹介します。

### 1. module × moduleResolution の不整合

一番事故が多いのがここです。

- `module: "commonjs"` + `moduleResolution: "bundler"` → **組み合わせ不可**。bundler は ESM 前提です
- `module: "node16"` を使うなら `moduleResolution` も `node16` に揃える必要があります
- `moduleResolution: "node"` は旧名称で、TS5では `node10` / `node16` / `bundler` の明示が推奨です

Vite や esbuild を使っているのに Node 向けの解決設定になっている（またはその逆）と、`Cannot find module` 系のエラーが「コードは正しいのに」発生します。

### 2. strict が切れている / noImplicitAny: false

`strict: true` は型安全の土台です。これが無いプロジェクトは、TypeScript を使っているのに any がどんどん混入して「ほぼ JavaScript」になりがちです。レガシーコードで一気に有効化できない場合は、`noImplicitAny` → `strictNullChecks` の順に段階導入するのがおすすめです。

### 3. paths を書いたのにランタイムが解決できない

`paths` で `@/*` エイリアスを設定しても、**tsc はコンパイル時にパスを書き換えません**。

- Vite → `vite-tsconfig-paths`
- Node 実行 → `tsx` や `tsconfig-paths`

など、ランタイム側の対応がセットで必要です。「エディタでは補完が効くのに実行すると落ちる」場合はほぼこれです。

### 4. esModuleInterop が無効

`import React from 'react'` のような default import で `TS1259` 系のエラーが出る典型原因です。CommonJS 資産と共存するなら `esModuleInterop: true` が無難です。

### 5. target が es5 のまま

現行ブラウザや Node 18+ を対象にするなら `es2020`〜`es2022` で問題ありません。古い環境向けの変換はバンドラー（の polyfill 設定）に任せ、tsc は型チェックに集中させる構成が主流です。

## スコアの目安

ツールでは減点方式で100点満点のスコアを出しています。

- **85点以上**: 健康。細かい改善のみ
- **60〜84点**: おおむね良好。warn項目の修正を推奨
- **59点以下**: module解決まわりに地雷がある可能性が高い

手元のプロジェクトでいくつか試すと、コピペ元の年代がだいたい分かって面白いです（`target: es5` + `moduleResolution` 未指定はだいたい2019年以前の記事由来…）。

## おまけ: 同じ場所に他のツールも置いてあります

[DevToolBox](https://1280itsuya.github.io/devtools/) という置き場に、開発中によく使う小物ツールをまとめています。

- [エラーメッセージ検索](https://1280itsuya.github.io/devtools/error/) — 実在の開発エラー238種を収録。エラー文を貼ると原因と直し方を即表示
- [JSON整形・構文チェック](https://1280itsuya.github.io/devtools/json/) — エラーの行・列を特定、末尾カンマの自動修正
- [Cron式チェッカー](https://1280itsuya.github.io/devtools/cron/) — GitHub ActionsのUTC→日本時間変換に対応

どれもブラウザ内で完結するので、社内コードや機密トークンでも安心して使えます。

「この設定もチェックしてほしい」という要望があればコメントで教えてください。診断ルールは随時追加しています。
