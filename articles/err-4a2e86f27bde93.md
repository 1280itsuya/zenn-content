---
title: "TS7006: Parameter 'x' implicitly has an 'any' type."
emoji: "🐛"
type: "tech"
topics: ["TypeScript", "型", "エラー解決"]
published: true
---

## 発生条件

- `tsconfig.json` で `"noImplicitAny": true`（または `"strict": true`）が有効な状態で、関数の引数に型注釈を書かなかった場合
- コールバック関数や無名関数の引数を型推論が確定できない文脈で使用した場合
- 型定義ファイルが存在しないサードパーティライブラリの関数に型なしのコールバックを渡した場合

## 原因

`TS7006: Parameter 'x' implicitly has an 'any' type.` は、[TypeScript](https://www.amazon.co.jp/s?k=TypeScript%20%E6%9C%AC&tag=1280itsuya22-22) が引数の型を文脈から推論できないときに発生する。`noImplicitAny` フラグが有効だと、暗黙的な `any` は型安全性を損なうとしてコンパイルエラーになる。

```typescript
// エラー: Parameter 'value' implicitly has an 'any' type.
function double(value) {
  return value * 2;
}

// コールバックでも同様に発生する
const nums = [1, 2, 3];
const result = nums.reduce((acc, cur) => acc + cur, 0); // これは推論できるが…

// 型定義のないオブジェクトを受け取るケース
const handler = {
  process: function(data) { // ← ここで TS7006
    console.log(data);
  }
};
```

TypeScript は `value` や `data` がどんな型であるか文脈から判断できないため、暗黙的に `any` と扱おうとする。`noImplicitAny` がこれを禁止するのでエラーになる。

## 修正方法

引数に明示的な型注釈を付けることで解消できる。

```typescript
// 修正: 引数に明示的な型を付ける
function double(value: number): number {
  return value * 2;
}

// オブジェクトメソッドのコールバックも同様に修正
type Payload = {
  id: number;
  name: string;
};

const handler = {
  process: function(data: Payload): void {
    console.log(data);
  }
};

// 型が複数ありうる場合はユニオン型を使う
function printValue(value: string | number): void {
  console.log(value);
}

// 本当に任意の型を受け取りたい場合は unknown を使い、内部で絞り込む
function inspect(value: unknown): void {
  if (typeof value === 'string') {
    console.log(value.toUpperCase());
  }
}
```

`any` を付ければコンパイルエラーは消えるが、型チェックが無効化されるため避けるべきだ。実際に受け取る型が決まっているなら具体的な型を、決まっていないなら `unknown` を使い、型ガードで絞り込むのが正しい対処になる。

`noImplicitAny` の目的は「型が曖昧なまま通過するコードを早期に検出する」ことにあるため、エラーを抑制するのではなく、引数の型を設計として明確にすることを優先してほしい。

## 似たエラーの解決記事

- [TS2531: Object is possibly 'null'.](https://zenn.dev/articles/err-1d6d240c665674)


---

📌 このエラーの要点だけ知りたい方は [エラー即答ページ](https://1280itsuya.github.io/devtools/error/e/bbcab74327c7/) へ。別のエラーで困ったら [エラーメッセージ検索ツール](https://1280itsuya.github.io/devtools/error/)（貼るだけで原因と直し方を表示）も使えます。


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/aikit260630)- 原因を根本から理解するなら体系的な技術書（Amazon: [TypeScript 型 実践](https://www.amazon.co.jp/s?k=TypeScript%20%E5%9E%8B%20%E5%AE%9F%E8%B7%B5&tag=1280itsuya22-22)）
