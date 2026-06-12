---
title: "TS2322: Type 'string' is not assignable to type 'number'."
emoji: "🐛"
type: "tech"
topics: ["TypeScript", "型", "エラー解決"]
published: true
---

## 発生条件

- `number` 型の変数やプロパティに、`string` 型の値をそのまま代入したとき
- `input.value` や `JSON.parse` の結果など、文字列で入ってくる値を数値型の変数へ渡したとき
- 関数の引数・戻り値・オブジェクトのプロパティで、宣言上は `number` なのに文字列を渡したとき

## 原因

`TS2322: Type 'string' is not assignable to type 'number'.` は、代入先が `number` 型なのに、代入しようとした値が `string` 型であることを示します。[TypeScript](https://www.amazon.co.jp/s?k=TypeScript%20%E6%9C%AC&tag=1280itsuya22-22) は型を自動で数値へ変換しないため、文字列をそのまま数値スロットへ入れるとコンパイル時に弾かれます。

最小再現コードは次のとおりです。

```typescript
interface User {
  age: number;
}

const ageInput: string = "25";

// Type 'string' is not assignable to type 'number'.
const user: User = {
  age: ageInput,
};
```

`ageInput` は `string`、`user.age` は `number` なので、宣言された型が一致せずエラーになります。HTML フォームの `<input>` から取得した値が `string` のまま流れてくるケースが典型です。

## 修正方法

値を明示的に数値へ変換してから代入します。文字列を数値化する場合は `Number()` を使います。

```typescript
interface User {
  age: number;
}

const ageInput: string = "25";

const user: User = {
  age: Number(ageInput), // string -> number へ変換
};

// 変換が失敗すると NaN になるため検証する
if (Number.isNaN(user.age)) {
  throw new Error("age が数値に変換できません");
}
```

要点は次の3つです。

- `Number(ageInput)` で `string` を `number` に変換し、代入先の型と一致させる。これで `is not assignable to type 'number'` は解消します。
- 整数だけが必要なら `parseInt(ageInput, 10)` も使えます。基数 `10` を必ず指定してください。
- `Number()` や `parseInt()` は変換に失敗すると `NaN` を返すため、`Number.isNaN()` で検証し、不正な入力を弾いておくと安全です。

なお、そもそも値が本来 `number` であるべきなのに `string` として宣言・取得している場合は、変換ではなく取得元の型定義を `number` に直すほうが根本的です。「変換して合わせる」か「型定義を正す」かは、その値の本来の意味で判断してください。

---

📌 このエラーの要点だけ知りたい方は [エラー即答ページ](https://1280itsuya.github.io/devtools/error/e/cc2daffdf766/) へ。別のエラーで困ったら [エラーメッセージ検索ツール](https://1280itsuya.github.io/devtools/error/)（貼るだけで原因と直し方を表示）も使えます。


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/aikit260612)- 原因を根本から理解するなら体系的な技術書（Amazon: [TypeScript 型 実践](https://www.amazon.co.jp/s?k=TypeScript%20%E5%9E%8B%20%E5%AE%9F%E8%B7%B5&tag=1280itsuya22-22)）
