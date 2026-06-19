---
title: "error[E0382]: borrow of moved value"
emoji: "🐛"
type: "tech"
topics: ["Rust", "エラー解決"]
published: true
---

## 発生条件

- 変数の所有権が別の変数や関数に**ムーブ**された後、元の変数を再度使おうとした時
- `String`・`Vec`・独自構造体など、`Copy` トレイトを実装していない型を複数箇所で使おうとした時
- クロージャやイテレータに値を `move` で渡した後、外側のスコープでその値を参照した時

## 原因

`error[E0382]: borrow of moved value` は、Rust の所有権ルール「値の所有者は常に1つ」に違反した時に発生する。

```rust
fn main() {
    let s = String::from("hello");
    let s2 = s;          // 所有権が s2 にムーブされる
    println!("{}", s);   // error[E0382]: borrow of moved value: `s`
}
```

上記では `let s2 = s;` の時点で `s` の所有権は `s2` に移る。その後 `s` を `println!` に渡そうとすると、コンパイラは「`s` はすでにムーブ済みの値であり、借用できない」として `E0382` を出す。

関数引数の場合も同様に起きる。

```rust
fn consume(s: String) {
    println!("{}", s);
}

fn main() {
    let s = String::from("hello");
    consume(s);          // ここで所有権がムーブ
    println!("{}", s);   // error[E0382]: borrow of moved value: `s`
}
```

`String` は `Copy` トレイトを実装していないため、代入や関数呼び出しのたびに所有権が移動する。

## 修正方法

状況に応じて3つのアプローチがある。

**1. 参照(`&`)で借用する（最も推奨）**

```rust
fn print_str(s: &String) {
    println!("{}", s);
}

fn main() {
    let s = String::from("hello");
    print_str(&s);       // 参照を渡すので所有権はムーブしない
    println!("{}", s);   // 問題なく使える
}
```

関数が値を所有する必要がない場合は、引数を `&String` や `&str` にするだけで解決する。

**2. `clone()` で明示的にコピーする**

```rust
fn main() {
    let s = String::from("hello");
    let s2 = s.clone();  // ヒープデータを含め完全にコピー
    println!("{}", s);   // s の所有権は残ったまま
    println!("{}", s2);
}
```

`clone()` はヒープのアロケーションを伴うため、ループ内など頻度が高い箇所では借用を優先する。

**3. `Copy` トレイトを持つ型を使う、または自作型に `Copy` を導出する**

```rust
#[derive(Debug, Clone, Copy)]
struct Point {
    x: f64,
    y: f64,
}

fn main() {
    let p = Point { x: 1.0, y: 2.0 };
    let p2 = p;          // Copy なので所有権ではなくビットコピーが走る
    println!("{:?}", p); // 問題なし
    println!("{:?}", p2);
}
```

`i32` や `f64` などのプリミティブ型はデフォルトで `Copy` を実装しているため `E0382` は起きない。ヒープを持たない小さな構造体なら `#[derive(Clone, Copy)]` を付けることで同じ挙動にできる。

どのアプローチを選ぶかは「呼び出し先が値を所有する必要があるか」で判断する。所有が不要なら借用、独立したコピーが必要なら `clone()`、型設計の段階で `Copy` の付与が適切かを検討する、という順序で考えると整理しやすい。

## 似たエラーの解決記事

- [error[E0499]: cannot borrow as mutable more than once at a time](https://zenn.dev/articles/err-e7d0176f3b29cf)


---

📌 このエラーの要点だけ知りたい方は [エラー即答ページ](https://1280itsuya.github.io/devtools/error/e/313db11e8234/) へ。別のエラーで困ったら [エラーメッセージ検索ツール](https://1280itsuya.github.io/devtools/error/)（貼るだけで原因と直し方を表示）も使えます。


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/aikit260619)- 原因を根本から理解するなら体系的な技術書（Amazon: [Rust 実践 入門](https://www.amazon.co.jp/s?k=Rust%20%E5%AE%9F%E8%B7%B5%20%E5%85%A5%E9%96%80&tag=1280itsuya22-22)）
