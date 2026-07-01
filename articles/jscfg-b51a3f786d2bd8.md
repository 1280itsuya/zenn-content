---
title: "next.config.js rewrites効かない"
emoji: "⚙️"
type: "tech"
topics: ["JavaScript", "TypeScript", "設定"]
published: true
---

## 発生条件

- `next.config.js` の `rewrites` を設定したにもかかわらず、リクエストが書き換えられずに 404 や意図しないルートへ到達する
- Next.js 13 以降の App Router 環境で、`/api/proxy` などのパスをバックエンドへ転送しようとしている
- `rewrites` の返り値が配列ではなく、またはオブジェクトの形式を誤って定義している

## 原因

最もよくある原因は、`rewrites` が非同期関数として定義されていない、または戻り値の形式が誤っているケースです。

```js
// ❌ 壊れている例
module.exports = {
  rewrites: [
    {
      source: '/api/:path*',
      destination: 'https://backend.example.com/:path*',
    },
  ],
}
```

上記は `rewrites` に配列をそのまま代入していますが、Next.js は `rewrites` を**非同期関数**として要求します。関数ではなく配列を渡した場合、Next.js はこのフィールドを無視し、書き換えが一切機能しません。エラーも出ないため気づきにくい点が厄介です。

また、以下のように関数にしていても `async` を付け忘れたり、`Promise` を返さない形になっていると同様に動作しません。

```js
// ❌ asyncを付け忘れた例
module.exports = {
  rewrites() {
    return [
      {
        source: '/api/:path*',
        destination: 'https://backend.example.com/:path*',
      },
    ]
  },
}
```

`rewrites()` は同期関数でも動作するケースはありますが、公式ドキュメントでは `async` 関数が前提とされており、将来的な互換性のためにも非同期で定義するべきです。

さらに、App Router 環境では `next.config.js` ではなく `next.config.mjs` を使用しているプロジェクトで、`module.exports` と `export default` を混在させてしまいエラーになるケースもあります。

## 修正方法

```js
// ✅ 正しい例（next.config.js）
/** @type {import('next').NextConfig} */
const nextConfig = {
  async rewrites() {
    return [
      {
        source: '/api/:path*',
        destination: 'https://backend.example.com/:path*',
      },
    ]
  },
}

module.exports = nextConfig
```

```mjs
// ✅ ESM形式の場合（next.config.mjs）
/** @type {import('next').NextConfig} */
const nextConfig = {
  async rewrites() {
    return [
      {
        source: '/api/:path*',
        destination: 'https://backend.example.com/:path*',
      },
    ]
  },
}

export default nextConfig
```

要点は以下の通りです。

- `rewrites` は必ず `async` 関数として定義し、配列を `return` で返す
- `next.config.js`（CommonJS）では `module.exports`、`next.config.mjs`（ESM）では `export default` を使う。混在させない
- 設定変更後は必ず開発サーバーを再起動する（`next dev` を `Ctrl+C` で止めて再実行）。ホットリロードでは `next.config.js` の変更が反映されない
- `:path*` のワイルドカードを使う場合、`source` と `destination` の両方に同じパラメータ名を揃える必要がある

設定を修正してもまだ動かない場合は、`.next` ディレクトリを削除してからビルドし直すと、キャッシュ起因の問題を排除できます。