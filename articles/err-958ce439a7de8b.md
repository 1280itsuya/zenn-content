---
title: "TS2345: Argument of type 'X' is not assignable to parameter of type 'Y"
emoji: "🐛"
type: "tech"
topics: ["TypeScript", "型", "エラー解決"]
published: true
---

## 発生条件

- 関数やメソッドに渡す引数の型が、シグネチャで要求している型と一致しない
- `string` を期待する引数に `number` や `undefined` を渡すなど、型が互換性を持たない
- ユニオン型や型エイリアスの絞り込みが不十分なまま関数に値を渡している

## 原因

`TS2345: Argument of type 'X' is not assignable to parameter of type 'Y'.` は、関数の呼び出し側が渡した値の型と、関数定義側が要求する型の間に互換性がない場合に発生する。

以下のような最小コードで再現できる。

```typescript
function greet(name: string): string {
  return `Hello, ${name}`;
}

const userId: number = 42;
greet(userId);
// TS2345: Argument of type 'number' is not assignable to parameter of type 'string'.
```

`greet` は `string` を要求しているが、渡した `userId` は `number` 型のため、[TypeScript](https://www.amazon.co.jp/s?k=TypeScript%20%E6%9C%AC&tag=1280itsuya22-22) がエラーを報告する。ユニオン型がからむケースも多い。

```typescript
function processStatus(status: "active" | "inactive"): void {
  console.log(status);
}

const raw: string = "active";
processStatus(raw);
// TS2345: Argument of type 'string' is not assignable to parameter of type '"active" | "inactive"'.
```

`raw` は `string` 型として推論されており、リテラル型のユニオンへの代入が拒否される。`string` は `"active" | "inactive"` より広い型なので、TypeScript は安全性のためにエラーとする。

## 修正方法

### パターン1: 型を合わせて渡す

```typescript
function greet(name: string): string {
  return `Hello, ${name}`;
}

const userId: number = 42;
greet(String(userId)); // number → string に変換して渡す
```

関数シグネチャが要求する型に合わせて値を変換する。`String()` や `.toString()` で明示的に変換するのが素直な対処。

### パターン2: `as const` またはリテラル型アサーションを使う

```typescript
function processStatus(status: "active" | "inactive"): void {
  console.log(status);
}

const raw = "active" as const; // 型が "active" に絞られる
processStatus(raw);
```

変数宣言時に `as const` を付けることで、`string` ではなくリテラル型 `"active"` として推論させる。

### パターン3: 型ガードで絞り込む

```typescript
function processStatus(status: "active" | "inactive"): void {
  console.log(status);
}

function isValidStatus(s: string): s is "active" | "inactive" {
  return s === "active" || s === "inactive";
}

const raw: string = getStatusFromSomewhere();

if (isValidStatus(raw)) {
  processStatus(raw); // ここでは "active" | "inactive" に絞られている
}
```

外部から来た文字列など、値が実行時まで確定しない場合は型ガードで絞り込んでから渡す。`as` キャストで強制的に回避する方法もあるが、実際に想定外の値が流れてきたときのバグを隠すため、型ガードを優先する。

`TS2345` は TypeScript が型の整合性を保証するための重要なエラーなので、`any` キャストで黙らせるのではなく、呼び出し側の型を関数シグネチャに合わせる形で解決するのが基本方針。

## 似たエラーの解決記事

- [TS2531: Object is possibly 'null'.](https://zenn.dev/articles/err-1d6d240c665674)


---

📌 このエラーの要点だけ知りたい方は [エラー即答ページ](https://1280itsuya.github.io/devtools/error/e/fc06ae10607d/) へ。別のエラーで困ったら [エラーメッセージ検索ツール](https://1280itsuya.github.io/devtools/error/)（貼るだけで原因と直し方を表示）も使えます。


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/aikit260630)- 原因を根本から理解するなら体系的な技術書（Amazon: [TypeScript 型 実践](https://www.amazon.co.jp/s?k=TypeScript%20%E5%9E%8B%20%E5%AE%9F%E8%B7%B5&tag=1280itsuya22-22)）
