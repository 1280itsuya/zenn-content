---
title: "You're importing a component that needs useState. It only works in a C"
emoji: "🐛"
type: "tech"
topics: ["Next.js", "React", "フロントエンド", "エラー解決"]
published: true
---

## 発生条件

- Next.js App Router でサーバーコンポーネント（デフォルト）のファイルに `useState` や `useEffect` などの [React](https://www.amazon.co.jp/s?k=React%20%E5%85%A5%E9%96%80%20%E6%9C%AC&tag=1280itsuya22-22) フックを直接記述したとき
- `"use client"` ディレクティブを書かずに、イベントハンドラや状態管理を含むコンポーネントを `app/` ディレクトリに配置したとき
- クライアント専用のサードパーティコンポーネントを、サーバーコンポーネントの中でそのまま import したとき

## 原因

App Router では `app/` 配下のファイルはデフォルトで**サーバーコンポーネント**として扱われる。サーバーコンポーネントはサーバー側でのみ実行されるため、ブラウザの状態に依存する `useState` を使えない。これが `You're importing a component that needs useState. It only works in a Client Component` というエラーの正体だ。

最小再現コードを示す。

```tsx
// app/counter.tsx  ← "use client" がない → サーバーコンポーネント扱い
import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0); // ❌ ここでエラー

  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}
```

このファイルを `app/page.tsx` などから import するだけで、ビルド時または開発サーバー起動時にエラーが発生する。

## 修正方法

ファイルの先頭に `'use client'` を追加するだけで解決する。これによりそのファイルとその依存ツリーがクライアントバンドルに含まれるようになる。

```tsx
// app/counter.tsx
'use client'; // ✅ 先頭に追加

import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}
```

要点を整理する。

- `'use client'` はファイルの**最先頭**（import より上）に書く必要がある
- `'use client'` を付けたコンポーネントは、そこから import される子コンポーネントも自動的にクライアントコンポーネントとして扱われる
- サーバーコンポーネントとクライアントコンポーネントを分離したい場合は、`useState` を使う部分だけを別ファイルに切り出して `'use client'` を付け、サーバーコンポーネントからそのファイルを import する構成が推奨される

```tsx
// app/page.tsx（サーバーコンポーネントのまま）
import Counter from './counter'; // クライアントコンポーネントを import するのはOK

export default function Page() {
  return (
    <main>
      <h1>カウンター</h1>
      <Counter />
    </main>
  );
}
```

サーバーコンポーネントはデータフェッチやレイアウト、クライアントコンポーネントはインタラクションと役割を分けることで、App Router の設計意図に沿った実装になる。

---

📌 このエラーの要点だけ知りたい方は [エラー即答ページ](https://1280itsuya.github.io/devtools/error/e/f8c5524bbcb3/) へ。別のエラーで困ったら [エラーメッセージ検索ツール](https://1280itsuya.github.io/devtools/error/)（貼るだけで原因と直し方を表示）も使えます。


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/aikit260617)- 原因を根本から理解するなら体系的な技術書（Amazon: [Next.js 実践](https://www.amazon.co.jp/s?k=Next.js%20%E5%AE%9F%E8%B7%B5&tag=1280itsuya22-22)）
