---
title: "【2026年最新】AI副業ツールが「TypeError: fetch is not a function」で動かない実例5パターン｜Node/ブラウザ環境の切り分けデバッグ手順"
emoji: "🛠"
type: "tech"
topics: ["javascript","nodejs","typescript","githubactions"]
published: true
---
> 本記事は筆者がQiitaで公開した記事のミラーです（原文: https://qiita.com/1280itsuya/items/aface3d400aab421891a ）。

ClaudeのAPIで記事生成バッチを回す、Pythonで吐いたデータをブラウザの管理画面で表示する——そういう「自前のAI副業ツール」が、コードは正しいのにJavaScriptの設定だけで動かない。この記事を読むと、`ERR_REQUIRE_ESM`・`fetch is not a function`・`process is not defined` といった頻出エラーを**5分で原因特定して直せる**ようになります。サンプルは全部コピペで動きます。

## 結論：9割は「Nodeか?ブラウザか?」「ESMかCommonJSか?」の2軸で切れる

先に答えを置きます。AI副業ツールが設定で死ぬとき、原因はだいたい次の2軸のどこかです。

1. **実行環境がNode.jsかブラウザか**（`window`/`document` があるのは後者だけ、`process`/`fs` があるのは前者だけ）
2. **モジュール方式がESM(`import`)かCommonJS(`require`)か**（`package.json` の `"type"` で決まる）

この2軸を最初に確定させるだけで、後述の5パターンは機械的に切り分けられます。まずは自分のツールがどっちで動いているかを1コマンドで確認しましょう。

```js
// env-probe.js — 自分のツールがどの環境で走っているか即判定する
function detectRuntime() {
  const result = {
    isNode: typeof process !== 'undefined' && !!process.versions?.node,
    isBrowser: typeof window !== 'undefined' && typeof document !== 'undefined',
    hasFetch: typeof fetch === 'function',
    nodeVersion: (typeof process !== 'undefined' && process.versions?.node) || null,
    moduleKind: typeof require === 'function' ? 'CommonJS(またはバンドラ)' : 'ESM(またはブラウザ)',
  };
  console.table(result);
  return result;
}

detectRuntime();
```

Nodeなら `node env-probe.js`、ブラウザなら `<script>` で読み込んでDevToolsのConsoleを見る。`isNode: true / hasFetch: false` と出たら、それだけでパターン1が確定します。

## パターン1：Node.js 16で `fetch is not a function`（Claude API呼び出しが即死）

一番多いのがこれ。`anthropic` のSDKを使わず、軽くしたくて素の `fetch` でClaude APIを叩くコードを書いたとき、ローカルのNodeが古いと落ちます。

グローバル `fetch` がNode.jsに標準搭載されたのは **Node 18**（v18でexperimental、**v21で安定化**）です。Node 16以前には存在しません。つまりエラーは「コードのミス」ではなく「Nodeのバージョン」が原因です。

```js
// claude-call.js — Node 18+ なら追加パッケージなしで動くClaude呼び出し
async function askClaude(prompt) {
  if (typeof fetch !== 'function') {
    throw new Error(
      `fetchが無い。Node ${process.versions.node} は18未満です。nvmで上げてください`
    );
  }
  const res = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'x-api-key': process.env.ANTHROPIC_API_KEY,
      'anthropic-version': '2023-06-01',
      'content-type': 'application/json',
    },
    body: JSON.stringify({
      model: 'claude-opus-4-8',
      max_tokens: 1024,
      messages: [{ role: 'user', content: prompt }],
    }),
  });
  if (!res.ok) {
    // 401ならキー、429ならレート、それ以外は本文を見る
    throw new Error(`Claude API ${res.status}: ${await res.text()}`);
  }
  const data = await res.json();
  return data.content[0].text;
}

askClaude('一文で自己紹介して').then(console.log);
```

**直し方の優先順位**：(1) `node -v` で確認 → (2) `nvm install 20 && nvm use 20` でNode 20系へ → (3) どうしても古いNodeに縛られるなら `npm i node-fetch` して `const fetch = (...a) => import('node-fetch').then(({default: f}) => f(...a))` で後付け。先頭の `typeof fetch !== 'function'` ガードを入れておくと、原因が一行で分かるのでデバッグが一瞬で終わります。

## パターン2：`require() of ES Module not supported`（自動投稿スクリプトが起動すらしない）

QiitaやZennへ自動投稿する系のスクリプトでよく踏みます。`package.json` に `"type": "module"` を書いた状態で、古い記事のコピペで `const fs = require('fs')` を混ぜると即死。逆に `"type"` 無し（=CommonJS扱い）で `import` を書いても `Cannot use import statement outside a module` で死にます。

判定は機械的です。

- `package.json` に `"type": "module"` あり → ファイル全体を `import`/`export` で統一。`require` 禁止。
- `"type"` 無し or `"commonjs"` → `require`/`module.exports` で統一。`import` 禁止。
- どうしても1ファイルだけ方式を変えたい → 拡張子を `.mjs`(ESM) / `.cjs`(CommonJS) にする。

ESM環境でCommonJS時代の便利関数（`__dirname` など）が無くて詰まるのもこのパターンの派生です。ESMには `__dirname` がありません。移植コードはこう書き換えます。

```js
// path-fix.mjs — ESMで __dirname が無いときの定番置き換え
import { fileURLToPath } from 'node:url';
import { dirname, join } from 'node:path';
import { readFileSync } from 'node:fs';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

// data/prompt.txt をスクリプトからの相対パスで安全に読む
const prompt = readFileSync(join(__dirname, 'data', 'prompt.txt'), 'utf-8');
console.log('読めた:', prompt.length, '文字');
```

移植時に `import.meta.url` を `require('url')` と混在させると振り出しに戻るので、**1ファイル内で方式を絶対に混ぜない**のが鉄則です。

## パターン3：ブラウザ管理画面で `process is not defined`（APIキーをフロントに置こうとした事故）

Pythonで生成したデータを、簡単な管理画面（素のHTML+JS）で表示する。ここで `process.env.ANTHROPIC_API_KEY` をそのまま書くと、ブラウザには `process` が存在しないので `Uncaught ReferenceError: process is not defined` で画面が真っ白になります。

そして重要なのは、**これはエラーで止まってくれるだけマシ**という点です。Viteなどのバンドラを使っていると `import.meta.env.VITE_ANTHROPIC_API_KEY` が**ビルド時に文字列へ展開され**、APIキーが配信JSにそのまま埋め込まれます。エラーは出ないのに、ブラウザのSourcesタブからキーが丸見えになる——これは事故です。

**原則：AI APIのシークレットを使う通信はブラウザから直接叩かない。** 必ずNode側（サーバー or GitHub Actions）を一枚挟みます。フロントは自前のエンドポイントだけを呼ぶ。

```js
// 悪い例（ブラウザ）: キーが配信JSに埋まる / CORSでも弾かれる
// fetch('https://api.anthropic.com/...', { headers: { 'x-api-key': KEY }})

// 良い例（ブラウザ）: 自前のサーバー経由にする。キーはサーバーだけが持つ
async function generate(prompt) {
  const res = await fetch('/api/generate', {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ prompt }),
  });
  if (!res.ok) throw new Error(`サーバー側エラー ${res.status}`);
  return (await res.json()).text;
}
```

ちなみにブラウザから直接Anthropic APIを叩くと、キー漏洩以前に**CORSで弾かれて** `Failed to fetch` になります。エラーメッセージが `process is not defined` でも `Failed to fetch` でも、原因は同じ「ブラウザからやってはいけないことをやっている」です。

## パターン4：`tsconfig.json` の `module` 不一致で、ビルドは通るのに実行で落ちる

TypeScriptでツールを書くと、`tsc` は静かに通るのに `node dist/index.js` で落ちる、という厄介なズレが起きます。原因は `tsconfig.json` の `module`/`moduleResolution` と、`package.json` の `"type"` が噛み合っていないこと。

ありがちな破綻：`tsconfig` で `"module": "ESNext"` を出力しているのに `package.json` に `"type": "module"` が無い → 出た `.js` は `import` 文を含むのにNodeがCommonJSとして読もうとして `Cannot use import statement outside a module`。

Node単体で動かすツールなら、まず迷ったらこの組み合わせが安定です。

```jsonc
// tsconfig.json（Node 20でCommonJS出力する最小・安定構成）
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "CommonJS",
    "moduleResolution": "Node",
    "esModuleInterop": true,   // import x from 'cjs-pkg' を許す
    "outDir": "dist",
    "strict": true
  }
}
```

このとき `package.json` には `"type": "module"` を**書かない**（=CommonJS）。「`tsconfig` の出力方式」と「`package.json` の `type`」を**必ずペアで意識する**——この一文だけ覚えれば、このパターンは二度と踏みません。

## パターン5：GitHub ActionsとローカルでNodeが違い、毎朝のバッチだけ落ちる

ローカルでは動くのに、毎朝7時に回しているGitHub Actionsのジョブだけ `fetch is not a function`（=パターン1の再来）で落ちる。原因は、ローカルがNode 20なのにActions側で `node-version` を指定し忘れ、ランナー既定のNodeで走っているケースです。環境を**明示的に固定**しないと、いつの間にかバージョンがずれます。

```yaml
# .github/workflows/daily-ai.yml — Nodeを固定して環境差をなくす
name: daily-ai-batch
on:
  schedule:
    - cron: '0 22 * * *'   # UTC22:00 = JST07:00
  workflow_dispatch:        # 手動実行ボタンも付けておくとデバッグが楽
jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'   # ← これを書かないと既定Nodeに引っ張られる
      - run: node -v          # ログにバージョンを残すと事故調査が一瞬
      - run: npm ci
      - run: node claude-call.js
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

ポイントは `node -v` をステップに1行入れておくこと。落ちたとき、ログの先頭を見れば「ローカルと違うバージョンで走っていた」が即わかります。`cron` のタイムゾーンはUTC固定なので、JST7時なら `0 22` です（ここを `0 7` にして「なぜか16時に動く」も定番の罠）。

## まとめ：エラーメッセージ別・原因の早見表

最後に、現場で迷わないための対応表です。

- `fetch is not a function`（Node）→ Node 18未満。バージョンを上げる（パターン1・5）
- `Cannot use import statement outside a module` → `package.json` の `"type"` と `import`/`require` 不一致（パターン2）
- `require() of ES Module not supported` → ESMパッケージを `require` した。`import` か `.mjs` へ（パターン2）
- `process is not defined`（ブラウザ）→ フロントにNode専用コード/APIキーを書いた。サーバーを挟む（パターン3）
- `Failed to fetch`（ブラウザ）→ CORS。ブラウザから外部AI APIを直叩きしている（パターン3）
- ビルドは通るのに実行で落ちる → `tsconfig` の `module` と `package.json` の `type` 不一致（パターン4）

冒頭の `env-probe.js` で「Nodeか/ブラウザか・ESMかCommonJSか」を先に確定させ、この早見表で引く。設定起因のエラーは、ほぼこの流れで詰みが解けます。AI副業ツールは「動かす環境を1か所に固定する」だけで、毎朝のバッチが静かに死ぬ事故の大半を消せます。
