---
title: "Cannot find name 'process'. Do you need @types/node?"
emoji: "🐛"
type: "tech"
topics: ["TypeScript", "VSCode", "開発環境"]
published: true
---

## 発生条件

- [TypeScript](https://www.amazon.co.jp/s?k=TypeScript%20%E6%9C%AC&tag=1280itsuya22-22) プロジェクトで `process.env.NODE_ENV` や `process.argv` など Node.js グローバルを参照したとき
- `@types/node` がインストールされていない、または `tsconfig.json` の `types` 配列に `"node"` が含まれていないとき
- Vite・Next.js・ts-node などのツールを素の `tsc` 環境と混在させていて型解決が部分的に外れているとき

## 原因

TypeScript は JavaScript 標準の型定義しか持たないため、Node.js 固有のグローバル（`process`、`Buffer`、`__dirname` など）は別パッケージ `@types/node` を通じて補完します。このパッケージが存在しない状態で `process` を参照すると、コンパイラは名前を解決できず次のエラーを出します。

```
Cannot find name 'process'. Do you need @types/node?
```

以下が最小の再現例です。

```typescript
// src/config.ts
const env = process.env.NODE_ENV; // ← ここで TS2304 が発生
console.log(env);
```

`@types/node` が未インストールの場合、`tsc` は `process` という名前を知らないため、上記1行だけでエラーになります。

## 修正方法

**ステップ1: パッケージをインストールする**

```bash
npm install --save-dev @types/node
```

yarn / pnpm の場合はそれぞれ以下のとおりです。

```bash
yarn add -D @types/node
pnpm add -D @types/node
```

**ステップ2: tsconfig.json を確認する**

`compilerOptions.types` を明示指定している場合、`"node"` を追加しないと型が読み込まれません。

```jsonc
// tsconfig.json（types を明示している場合のみ追記が必要）
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "strict": true,
    "types": ["node"] // ← "node" を追加
  }
}
```

`types` フィールドを書いていない場合は、`node_modules/@types` 以下のパッケージが自動的に全件読み込まれるため、ステップ1だけで `Cannot find name 'process'` は解消されます。

**ステップ3: 動作確認**

```bash
npx tsc --noEmit
```

エラーが消えたことを確認します。型チェックのみ実行する `--noEmit` を使うと出力ファイルを生成せず手早く確かめられます。

---

**よくある補足ポイント**

- `types` を明示しているプロジェクトでは、`@types/node` を入れても `tsconfig.json` を直さないと効きません。両方の確認が必要です。
- モノレポ構成の場合、ルートの `tsconfig` ではなくサブパッケージ側の `tsconfig` が使われていることがあるため、どの設定ファイルが有効かを `tsc --showConfig` で確認するのが確実です。
- `@types/node` のバージョンは利用している Node.js のメジャーバージョンと揃えると型の齟齬を防ぎやすくなります（例: Node 20 系なら `@types/node@20`）。

---

📌 このエラーの要点だけ知りたい方は [エラー即答ページ](https://1280itsuya.github.io/devtools/error/e/a45f9f598c61/) へ。別のエラーで困ったら [エラーメッセージ検索ツール](https://1280itsuya.github.io/devtools/error/)（貼るだけで原因と直し方を表示）も使えます。


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/aikit260624)- 原因を根本から理解するなら体系的な技術書（Amazon: [TypeScript 実践](https://www.amazon.co.jp/s?k=TypeScript%20%E5%AE%9F%E8%B7%B5&tag=1280itsuya22-22)）
