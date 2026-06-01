---
title: "第3章 dotenvでAPIキーがundefinedになる5パターンを自動検出"
free: false
---

## config()がimportより後で実行される順序バグを正規表現で検出

Node.js の `import` はファイル先頭へ巻き上げられるため、`require('dotenv').config()` を2行目に書いても、ESM の `import OpenAI from 'openai'` が先に評価され `process.env.OPENAI_API_KEY` が `undefined` になる。doctor.js では行番号を比較して順序違反を赤判定する。

```js
import { readFileSync } from 'node:fs';
export function checkConfigOrder(file = 'index.js') {
  const lines = readFileSync(file, 'utf8').split(/\r?\n/);
  const cfg = lines.findIndex(l => /dotenv.*config\(/.test(l));
  const imp = lines.findIndex(l => /^\s*import .*(openai|anthropic)/.test(l));
  if (imp !== -1 && cfg > imp)
    return { ok: false, fix: `① index.js の1行目を 'import "dotenv/config"' に変更` };
  return { ok: cfg !== -1 };
}
```

## .envのBOM・CRLF・全角スペースを走査するcheckEnv()

Windows のメモ帳で保存すると先頭3バイト `EF BB BF`（BOM付きUTF-8）が混入し、最初のキー名が `\uFEFFOPENAI_API_KEY` になって参照が外れる。CRLF と全角スペースも同時に走査する。

```js
export function checkEnv(path = '.env') {
  const raw = readFileSync(path, 'utf8');
  const issues = [];
  if (raw.charCodeAt(0) === 0xFEFF) issues.push('BOM付き→UTF-8(BOMなし)で再保存');
  if (/\r\n/.test(raw)) issues.push('CRLF→LFへ変換');
  if (/[　]/.test(raw)) issues.push('全角スペース混入');
  return { ok: issues.length === 0, fix: issues.join(' / ') };
}
```

## dotenv.config({path})で.envパス違いを潰す

`node scripts/run.js` のように作業ディレクトリが2階層下だと、デフォルトの `process.cwd()/.env` を探して空振りする。`path.resolve` でプロジェクトルートを明示すれば、どの階層から起動しても同じ `.env` を読む。

```js
import { resolve } from 'node:path';
import dotenv from 'dotenv';
dotenv.config({ path: resolve(import.meta.dirname, '../.env') });
console.log(process.env.OPENAI_API_KEY ? 'loaded' : 'NULL: path違い');
```

## .env.exampleとの差分で必須キー4個の欠損を一覧出力

本番デプロイ時に `.env` は `.gitignore` で除外され消える。`.env.example` をコミット済みの仕様書として扱い、必須キー配列との差分を出す。欠損が1個でもあれば赤。

```js
const REQUIRED = ['OPENAI_API_KEY', 'ANTHROPIC_API_KEY', 'PINTEREST_TOKEN', 'WP_APP_PASS'];
export function checkMissing() {
  const missing = REQUIRED.filter(k => !process.env[k]);
  return missing.length
    ? { ok: false, fix: `欠損${missing.length}個: ${missing.join(', ')}` }
    : { ok: true };
}
```

## KEY = value のスペース混入で3時間溶かした実例

`OPENAI_API_KEY = sk-xxx` と `=` の両側に半角スペースを入れると、dotenv は値を ` sk-xxx`（先頭スペース付き）として読み、API は `401 Invalid Authentication` を返す。エラー文に「undefined」と出ないため原因特定に3時間かかった。doctor.js で `KEY␣=` 形式を検出して即修正コマンドを出す。

```js
export function checkSpacing(path = '.env') {
  const bad = readFileSync(path, 'utf8').split(/\r?\n/)
    .filter(l => /^\w+\s+=|=\s/.test(l));
  return bad.length
    ? { ok: false, fix: `${bad.length}行で '=' 前後にスペース→詰める` }
    : { ok: true };
}
// 5関数を Promise.all で並走させ 緑/赤+fix を一括出力すれば 0.2秒で5パターン診断完了
```

<!-- topics: nodejs, playwright, dotenv, windows, automation -->
