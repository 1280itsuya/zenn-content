---
title: "【完全版】SyntaxError: Cannot use import statement outside a module"
emoji: "💎"
type: "tech"
topics: ["エラー解決"]
published: true
price: 300
---

## 何が起きているのか（原因の本質）

`SyntaxError: Cannot use import statement outside a module` は、Node.js が **CommonJS モジュールとして扱っているファイル内で ES Module (ESM) 構文 `import` を使った** ときに発生する。単なる書き方の問題ではなく、Node.js のモジュールシステムが二重仕様になっている構造的な問題だ。

Node.js はファイルをロードする際に「CJS か ESM か」を事前に決定する。判定ロジックは次のとおり。

1. ファイルの拡張子が `.mjs` → ESM として扱う
2. ファイルの拡張子が `.cjs` → CJS として扱う
3. 拡張子が `.js` → **最寄りの `package.json` の `"type"` フィールドを参照**
   - `"type": "module"` → ESM
   - `"type": "commonjs"` または未設定 → **CJS**

「未設定 = CJS」がデフォルトであるため、`package.json` に `"type"` を書いていない状態で `.js` ファイルに `import` を書くと、Node.js は CJS パーサーでファイルを読み込もうとし、`import` 文を解釈できずにエラーを投げる。

エラーが出るのはパース段階なので、実行されるコードが正しくても関係ない。Node.js がファイルを読み込んだ瞬間に弾かれる。

---

## なぜ多くの解説では解決しないのか

検索上位に出てくる解説の多くは「`package.json` に `"type": "module"` を追加しろ」か「ファイル拡張子を `.mjs` にしろ」で終わる。これ自体は間違いではないが、**プロジェクト構成が少し複雑になると効かない**ケースがある。

### ケース1: `package.json` が複数ある

モノレポや、`node_modules` 内にネストした `package.json` が存在する構成では、「最寄りの `package.json`」が自分が編集したものとは限らない。ルートに `"type": "module"` を追加しても、サブディレクトリに別の `package.json`（`"type"` 未設定）があれば、そのディレクトリ配下のファイルは依然として CJS 扱いになる。

### ケース2: トランスパイラ経由でエラーが変化する

ts-node や Babel を使っている場合、`SyntaxError: Cannot use import statement outside a module` は **ts-node/Babel の設定ミスが原因** であることが多い。`package.json` をいくら直しても解決しない。ts-node なら `tsconfig.json` の `"module": "CommonJS"` と ts-node の `esm: true` の整合が取れていないことが主因だ。

### ケース3: Jest や Vitest などのテストランナー

Jest はデフォルトで Babel 変換を挟むが、設定によっては ESM ファイルを変換せずに Node.js に渡す。この場合、本体コードは問題なく動くのに **テストだけでエラーが出る**。`--experimental-vm-modules` フラグや `transform` 設定を変えるだけで解決するが、「`package.json` を直せ」系の解説では一切言及されない。

### ケース4: `require()` と `import` の混在

`"type": "module"` を設定したあと、`require()` を使っている箇所が残っていると別のエラー（`ReferenceError: require is not defined`）が発生する。対症療法で `import` を `require` に書き換えるパターンを繰り返すと、混在状態が生まれてどちらのエラーも出る地獄になる。

---

## 完全な再現と修正コード

### 最小再現コード

以下のファイル構成で `SyntaxError: Cannot use import statement outside a module` を確実に再現できる。

```json
// package.json（"type" フィールドなし）
{
  "name": "repro",
  "version": "1.0.0"
}
```

```js
// utils.js
export function greet(name) {
  return `Hello, ${name}`;
}
```

```js
// index.js
import { greet } from './utils.js';
console.log(greet('world'));
```

```bash
node index.js
# => SyntaxError: Cannot use import statement outside a module
```

### 修正パターンA: `package.json` に `"type": "module"` を追加

プロジェクト全体を ESM に統一できる場合はこれが最もシンプル。

```json
// package.json
{
  "name": "repro",
  "version": "1.0.0",
  "type": "module"
}
```

```bash
node index.js
# => Hello, world
```

### 修正パターンB: 拡張子を `.mjs` に変更

`package.json` を変えたくない場合（他のファイルが CJS を前提にしている場合など）はファイル単位で制御する。

```bash
mv index.js index.mjs
mv utils.js utils.mjs
```

```js
// index.mjs
import { greet } from './utils.mjs'; // 拡張子まで明示する
console.log(greet('world'));
```

```bash
node index.mjs
# => Hello, world
```

### 修正パターンC: ts-node + TypeScript で ESM を使う場合

```json
// tsconfig.json
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "bundler",
    "target": "ES2022"
  }
}
```

```json
// package.json
{
  "type": "module",
  "scripts": {
    "dev": "node --loader ts-node/esm src/index.ts"
  }
}
```

`ts-node/esm` ローダーを明示しないと、ts-node は TypeScript を CJS としてコンパイルするため、`import` のままコードを書いていても実行時に CJS として扱われて同じエラーが出る。

### 修正パターンD: Jest で ESM を有効にする

```js
// jest.config.js
export default {
  extensionsToTreatAsEsm: ['.ts'],
  transform: {}
};
```

```bash
node --experimental-vm-modules node_modules/.bin/jest
```

---

## 本番投入前チェックリスト

- [ ] プロジェクトのすべての `package.json`（ネストも含む）で `"type"` フィールドが意図どおりに設定されているか確認する
- [ ] `import` と `require()` が同一ファイル内、または同一モジュールスコープ内で混在していないか確認する
- [ ] トランスパイラ（ts-node/Babel/esbuild）を使っている場合、その出力形式（CJS/ESM）と `package.json` の `"type"` が整合しているか確認する
- [ ] テストランナー（Jest/Vitest）が ESM ファイルを変換しているか、またはネイティブ ESM モードで動いているかを明確にする
- [ ] `node_modules` 内の依存パッケージが ESM のみ提供（dual package ではない）になっていないか確認する（`exports` フィールドで `require` エントリが存在するかチェック）
- [ ] CI 環境と開発環境の Node.js バージョンが揃っているか確認する（Node.js 12 以前は ESM サポートが実験的）
- [ ] `import()` 動的インポートを CJS ファイル内で使う場合、`async/await` でラップされているか確認する

---

## 関連する2つのエラーへの応用

### 1. `ReferenceError: require is not defined in ES module scope`

このエラーは `SyntaxError: Cannot use import statement outside a module` の**鏡像**だ。`"type": "module"` を設定したあとに、古い CommonJS スタイルの `require()` 呼び出しが残っていると発生する。

本記事の考え方の転用: Node.js がモジュールタイプを「事前決定」するという仕組みを理解していれば、`require` が使えないのも当然だとわかる。ESM スコープでは `require` 関数自体が存在しない。解決策は同じで「`import` に書き換える」か「`.cjs` 拡張子にして CJS に戻す」か「`createRequire` を使って ESM 内で `require` を再現する」の三択。修正方向の判断軸は「このファイル全体を ESM にしたいか CJS にしたいか」という一点であり、それは本記事と変わらない。

### 2. `ERR_REQUIRE_ESM: require() of ES Module ... not supported`

外部パッケージが ESM のみで配布されている（`exports` フィールドに `require` エントリがない）場合、CJS プロジェクトからそのパッケージを `require()` しようとすると発生する。`node-fetch` v3 や `chalk` v5 などが代表例。

本記事の考え方の転用: 根本原因は「CJS が ESM を同期的に `require` できない」という Node.js の制約だ。CJS から ESM をロードするには非同期の `import()` を使う必要がある。本記事で学んだ「CJS と ESM の境界はファイル単位で決まる」という知識があれば、「パッケージ側が ESM 境界の外にいる」と理解でき、解決策として `import()` による動的ロードか、当該パッケージの旧バージョン（CJS 対応版）を使うか、自プロジェクトを ESM に移行するかのいずれかを選べるようになる。

---

簡易版(無料)はこちら: [無料記事](https://zenn.dev/articles/err-85d63d44f92d4a)
