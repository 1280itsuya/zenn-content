---
title: "Too many re-renders. React limits the number of renders to prevent an "
emoji: "🐛"
type: "tech"
topics: ["React", "JavaScript", "フロントエンド", "エラー解決"]
published: true
---

## 発生条件

- レンダリング中に `setState` を直接呼び出している（`onClick={setState(value)}` のような形）
- `useEffect` の依存配列が不適切で、エフェクト内の更新が再レンダーを引き起こし続けている
- JSX の `onChange` / `onClick` 等に関数呼び出し式（`handler()`）を直接渡している

## 原因

[React](https://www.amazon.co.jp/s?k=React%20%E5%85%A5%E9%96%80%20%E6%9C%AC&tag=1280itsuya22-22) は `Too many re-renders. React limits the number of renders to prevent an infinite loop.` というエラーを、レンダリングフェーズ中に state の更新が連鎖して止まらないと検知した場合に投げます。

最も典型的なパターンは、イベントハンドラに関数の**呼び出し結果**を渡してしまうケースです。

```jsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  // NG: レンダリング時に setCount(count + 1) が即実行される
  return (
    <button onClick={setCount(count + 1)}>
      {count}
    </button>
  );
}
```

`onClick={setCount(count + 1)}` と書くと、JSX の評価時点（＝レンダリング中）に `setCount` が実行されます。その結果 state が変わり → 再レンダー → また `setCount` が実行される、という無限ループに入ります。

`useEffect` での同様の例：

```jsx
import { useState, useEffect } from 'react';

function Timer() {
  const [tick, setTick] = useState(0);

  // NG: tick が変わるたびに setTick が走り、tick がまた変わる
  useEffect(() => {
    setTick(tick + 1);
  }, [tick]);

  return <div>{tick}</div>;
}
```

こちらは依存配列に `tick` を含めているため、`setTick` → `tick` 更新 → エフェクト再実行 → `setTick` の無限ループになります。

## 修正方法

### イベントハンドラのケース

```jsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  // OK: アロー関数で包み、クリック時にだけ実行させる
  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}
```

`onClick={() => setCount(count + 1)}` とアロー関数で包むことで、クリックイベントが発生したときだけ `setCount` が呼ばれるようになります。

### useEffect のケース

```jsx
import { useState, useEffect } from 'react';

function Timer() {
  const [tick, setTick] = useState(0);

  // OK: 関数形式の更新 + 依存配列を空にする
  useEffect(() => {
    const id = setInterval(() => {
      setTick(prev => prev + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []); // 依存配列を空にしてマウント時のみ実行

  return <div>{tick}</div>;
}
```

ポイントは2つです。

1. **依存配列を適切に絞る** — 今回は初回マウント時だけ実行したいので `[]` にします。もし特定の値が変わったときだけ動かしたい場合は、その値だけを依存配列に入れてループが起きないか確認します。
2. **更新関数形式 `prev => prev + 1` を使う** — 現在の state 値を依存配列に入れずに参照できるため、不要な再実行を防ぎます。

`Too many re-renders` が出たら、まずレンダリングフェーズ中に `setState` 系の関数が即実行されていないか確認するのが最初の手順です。

---

📌 このエラーの要点だけ知りたい方は [エラー即答ページ](https://1280itsuya.github.io/devtools/error/e/0cdaa3375ea4/) へ。別のエラーで困ったら [エラーメッセージ検索ツール](https://1280itsuya.github.io/devtools/error/)（貼るだけで原因と直し方を表示）も使えます。


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/agentrules260615)- 原因を根本から理解するなら体系的な技術書（Amazon: [React 実践 入門](https://www.amazon.co.jp/s?k=React%20%E5%AE%9F%E8%B7%B5%20%E5%85%A5%E9%96%80&tag=1280itsuya22-22)）
