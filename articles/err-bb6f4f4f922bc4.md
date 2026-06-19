---
title: "error[E0308]: mismatched types"
emoji: "🐛"
type: "tech"
topics: ["Rust", "エラー解決"]
published: true
---

## 発生条件

- 関数の戻り値の型と、実際に返している値の型が一致しない場合
- 変数に代入しようとした値が、変数の型アノテーションと異なる場合
- `Option<T>` や `Result<T, E>` を期待している箇所に、内側の型 `T` をそのまま渡した場合（またはその逆）

## 原因

`error[E0308]: mismatched types` は、Rustのコンパイラが型推論の結果と実際の型が食い違っていると判断したときに発生します。

```rust
fn to_string(n: i32) -> String {
    n  // error[E0308]: mismatched types
       // expected `String`, found `i32`
}
```

```rust
fn first_char(s: &str) -> char {
    s.chars().next()  // error[E0308]: mismatched types
                      // expected `char`, found `Option<char>`
}
```

```rust
let x: u32 = -1_i32;  // error[E0308]: mismatched types
                       // expected `u32`, found `i32`
```

典型的なパターンは3つです。①整数・浮動小数点の符号あり/なし・ビット幅の違い、②`String` と `&str` の混在、③`Option`/`Result` のラッパーを剥がし忘れ（または逆に余計に包んでしまう）。

## 修正方法

### ① 数値型の変換

```rust
fn double(n: i32) -> i64 {
    n as i64 * 2  // as で明示的にキャスト
}

// または Into トレイトを使う
fn double_into(n: i32) -> i64 {
    let n: i64 = n.into();
    n * 2
}
```

`as` は数値型間のキャストに使えますが、符号付きから符号なしへのキャストは値が変わる場合があるため注意が必要です。安全性が気になる場合は `i32::try_into()` + `Result` で処理します。

### ② String と &str の変換

```rust
fn greet(name: &str) -> String {
    // &str → String は .to_string() または .to_owned()
    format!("Hello, {}!", name)
}

fn take_str(s: String) -> &'static str {
    // String → &str は & でスライスにする
    // ただしライフタイムに注意。ここでは静的文字列リテラルを返す例
    "static"
    // s.as_str() は返せない（s がスコープを抜けるため）
}
```

### ③ Option / Result のアンラップ見直し

```rust
fn first_char(s: &str) -> char {
    // NG: s.chars().next() は Option<char>
    // OK: unwrap_or でデフォルト値を与える
    s.chars().next().unwrap_or('\0')
}

fn parse_number(s: &str) -> Result<i32, _> {
    // NG: s.parse::<i32>() の結果をそのまま i32 として使う
    // OK: ? 演算子で Result を伝播させる
    let n: i32 = s.parse()?;
    Ok(n * 2)
}
```

`error[E0308]` が出たとき、エラーメッセージの `expected ... found ...` の行を読むだけで、どの型からどの型へ変換が必要かが分かります。`as` / `.into()` / `.parse()` / `?` 演算子のどれが適切かは、変換元と変換先の型の組み合わせで判断してください。なお `unwrap()` は `None` や `Err` でパニックするため、本番コードでは `unwrap_or` / `unwrap_or_else` / `?` の使用を検討してください。

## 似たエラーの解決記事

- [error[E0499]: cannot borrow as mutable more than once at a time](https://zenn.dev/articles/err-e7d0176f3b29cf)


---

📌 このエラーの要点だけ知りたい方は [エラー即答ページ](https://1280itsuya.github.io/devtools/error/e/577bb2de45a0/) へ。別のエラーで困ったら [エラーメッセージ検索ツール](https://1280itsuya.github.io/devtools/error/)（貼るだけで原因と直し方を表示）も使えます。


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/aikit260619)- 原因を根本から理解するなら体系的な技術書（Amazon: [Rust 実践 入門](https://www.amazon.co.jp/s?k=Rust%20%E5%AE%9F%E8%B7%B5%20%E5%85%A5%E9%96%80&tag=1280itsuya22-22)）
