---
title: "JSX element implicitly has type 'any'"
emoji: "🐛"
type: "tech"
topics: ["TypeScript", "VSCode", "開発環境"]
published: true
---

## 発生条件

- [TypeScript](https://www.amazon.co.jp/s?k=TypeScript%20%E6%9C%AC&tag=1280itsuya22-22) プロジェクトで `@types/react` がインストールされておらず、JSX 要素の型が解決できないとき
- `tsconfig.json` の `"jsx"` オプションが未設定、または `"preserve"` のままで [React](https://www.amazon.co.jp/s?k=React%20%E5%85%A5%E9%96%80%20%E6%9C%AC&tag=1280itsuya22-22) の型変換が走らないとき
- React 17 以降の新 JSX Transform を使っているが、`tsconfig.json` に `"jsx": "react-jsx"` が指定されていないとき

## 原因

`JSX element implicitly has type 'any'` は、TypeScript が JSX 要素を型解決しようとした際に、対応する型定義が見つからないために発生する。典型的な最小再現コードは以下のとおり。

```tsx
// @types/react が無い、または tsconfig の jsx 設定が不足している状態
function Greeting() {
  return <p>Hello</p>; // ← JSX element implicitly has type 'any'
}
```

TypeScript は JSX を `React.createElement` 呼び出し、あるいは新 JSX Transform の `_jsx` 関数へ変換する。しかし `@types/react` が存在しなければ `React.JSX.Element` 型が未定義となり、すべての JSX ノードが暗黙的に `any` になる。同様に `tsconfig.json` で `jsx` フィールドが適切に設定されていない場合も、同じエラーが出る。

```json
// 問題のある tsconfig.json（jsx 未設定 or 不適切）
{
  "compilerOptions": {
    "strict": true
    // "jsx" が無いか "preserve" のまま
  }
}
```

## 修正方法

**ステップ 1: `@types/react` を追加する**

```bash
npm install --save-dev @types/react @types/react-dom
```

React 本体がすでに入っている場合でも、型定義パッケージは別途必要。

**ステップ 2: `tsconfig.json` の `jsx` を正しく設定する**

React 17 以降（新 JSX Transform）を使う場合:

```json
{
  "compilerOptions": {
    "strict": true,
    "jsx": "react-jsx",
    "lib": ["dom", "es2020"],
    "moduleResolution": "bundler"
  }
}
```

React 16 以前、または `import React from 'react'` を明示する構成の場合は `"jsx": "react"` を使う。

```json
{
  "compilerOptions": {
    "strict": true,
    "jsx": "react"
  }
}
```

修正後、先ほどのコードは以下のように型が正しく解決される。

```tsx
import { FC } from 'react';

const Greeting: FC = () => {
  return <p>Hello</p>; // JSX element implicitly has type 'any' が消える
};

export default Greeting;
```

要点は 2 つだけ。`@types/react` を devDependencies に入れること、そして `tsconfig.json` の `"jsx"` を使用している React のバージョンと Transform モードに合わせて明示的に指定すること。どちらか片方だけでは `JSX element implicitly has type 'any'` が残るケースがあるため、両方セットで確認する。

## 似たエラーの解決記事

- [Cannot find name 'process'. Do you need @types/node?](https://zenn.dev/articles/err-dc67d8f6f7d6ea)


---

📌 このエラーの要点だけ知りたい方は [エラー即答ページ](https://1280itsuya.github.io/devtools/error/e/d631360b3381/) へ。別のエラーで困ったら [エラーメッセージ検索ツール](https://1280itsuya.github.io/devtools/error/)（貼るだけで原因と直し方を表示）も使えます。


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/aikit260624)- 原因を根本から理解するなら体系的な技術書（Amazon: [TypeScript 実践](https://www.amazon.co.jp/s?k=TypeScript%20%E5%AE%9F%E8%B7%B5&tag=1280itsuya22-22)）
