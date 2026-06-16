---
title: "React Hook useEffect has a missing dependency"
emoji: "🐛"
type: "tech"
topics: ["React", "JavaScript", "フロントエンド", "エラー解決"]
published: true
---

## 発生条件

- `useEffect` 内でコンポーネントのスコープに属する変数や関数を参照しているが、依存配列（第2引数）にそれらが含まれていない
- ESLint の `react-hooks/exhaustive-deps` ルールが有効な状態で、依存配列が不完全なコードを記述している
- `useEffect` 内でプロップス、ステート、またはコンポーネント内で定義された関数を使用しており、それらが依存配列に列挙されていない

## 原因

ESLint は `useEffect` 内で参照されるすべての外部値を依存配列に含めることを要求する。以下のコードでは `fetchData` がコンポーネント内で定義された関数であるにもかかわらず依存配列に含まれておらず、`[React](https://www.amazon.co.jp/s?k=React%20%E5%85%A5%E9%96%80%20%E6%9C%AC&tag=1280itsuya22-22) Hook useEffect has a missing dependency: 'fetchData'` という警告が発生する。

```tsx
import { useEffect, useState } from "react";

function UserList({ userId }: { userId: number }) {
  const [data, setData] = useState(null);

  // コンポーネント内で定義された関数
  const fetchData = async () => {
    const res = await fetch(`/api/users/${userId}`);
    setData(await res.json());
  };

  useEffect(() => {
    fetchData(); // ← fetchData が依存配列にない
  }, []); // ESLint: React Hook useEffect has a missing dependency: 'fetchData'

  return <div>{JSON.stringify(data)}</div>;
}
```

`fetchData` はレンダーのたびに新しい関数インスタンスとして再生成される。依存配列に追加すると毎レンダーで `useEffect` が再実行されてしまうため、単純に追加するだけでは無限ループになる場合がある。

## 修正方法

### パターン1: `useCallback` で関数を安定化する（推奨）

関数自体を `useCallback` でメモ化し、その関数を依存配列に含める。

```tsx
import { useCallback, useEffect, useState } from "react";

function UserList({ userId }: { userId: number }) {
  const [data, setData] = useState(null);

  const fetchData = useCallback(async () => {
    const res = await fetch(`/api/users/${userId}`);
    setData(await res.json());
  }, [userId]); // userId が変わったときだけ fetchData を再生成

  useEffect(() => {
    fetchData();
  }, [fetchData]); // fetchData を依存配列に追加

  return <div>{JSON.stringify(data)}</div>;
}
```

`useCallback` の依存配列に `userId` を含めることで、`userId` が変化したときのみ `fetchData` が再生成され、`useEffect` も再実行される。

### パターン2: 関数を `useEffect` 内部に移動する

依存配列の管理を簡単にしたい場合は、関数ごと `useEffect` の中に移す。

```tsx
import { useEffect, useState } from "react";

function UserList({ userId }: { userId: number }) {
  const [data, setData] = useState(null);

  useEffect(() => {
    const fetchData = async () => {
      const res = await fetch(`/api/users/${userId}`);
      setData(await res.json());
    };
    fetchData();
  }, [userId]); // userId だけを依存配列に列挙すればよい

  return <div>{JSON.stringify(data)}</div>;
}
```

関数を `useEffect` 内部に定義することで、ESLint は `userId` のみを依存として要求する。関数を外部に公開する必要がないケースではこちらがシンプル。

### パターン3: `setState` の関数形式で状態依存を回避する

更新対象が直前の状態値のみに依存する場合は、`setState` のコールバック形式を使うことで依存配列からステートを除外できる。

```tsx
useEffect(() => {
  const id = setInterval(() => {
    setCount((prev) => prev + 1); // count を依存配列に含めずに済む
  }, 1000);
  return () => clearInterval(id);
}, []); // 依存なしで安全
```

`React Hook useEffect has a missing dependency` という警告を ESLint の無効化コメントで黙らせるのは避けること。警告はバグの予兆であることが多く、上記いずれかのパターンで根本から対処するのが適切。

---

📌 このエラーの要点だけ知りたい方は [エラー即答ページ](https://1280itsuya.github.io/devtools/error/e/3e6b64c6e138/) へ。別のエラーで困ったら [エラーメッセージ検索ツール](https://1280itsuya.github.io/devtools/error/)（貼るだけで原因と直し方を表示）も使えます。


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/agentrules260615)- 原因を根本から理解するなら体系的な技術書（Amazon: [React 実践 入門](https://www.amazon.co.jp/s?k=React%20%E5%AE%9F%E8%B7%B5%20%E5%85%A5%E9%96%80&tag=1280itsuya22-22)）
