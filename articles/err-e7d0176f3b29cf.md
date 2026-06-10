---
title: "error[E0499]: cannot borrow as mutable more than once at a time"
emoji: "🐛"
type: "tech"
topics: ["Rust", "エラー解決"]
published: true
---

## 発生条件

- 同じ変数に対して `&mut` を取った状態のまま、もう一度 `&mut` を取ろうとしたとき
- メソッドの戻り値が `&mut self` を握り続けたまま、別の可変メソッドを呼んだとき
- `Vec` や `HashMap` の一要素を可変借用しながら、同じコレクション全体を可変操作したとき

いずれも `error[E0499]: cannot borrow as mutable more than once at a time` として現れます。Rust の借用規則では、ある値に対する可変借用は同時に1つしか存在できないためです。

## 原因

最小再現コードです。`first` が `v` への可変借用を保持したまま、`v.push` がもう一つの可変借用を要求しています。

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    let first = &mut v[0];   // 1つ目の可変借用
    v.push(4);               // 2つ目の可変借用 → E0499
    println!("{}", first);   // ここで first がまだ生きている
}
```

`first` の生存期間が最後の `println!` まで続くため、その間に発生する `v.push(4)` が "cannot borrow as mutable more than once" と判定されます。借用がいつまで生きているか、が判定の決め手です。

## 修正方法

1つ目の借用の使用を終わらせてから2つ目を始めるよう、スコープと順序を分けます。

```rust
fn main() {
    let mut v = vec![1, 2, 3];

    // 1つ目の可変借用はこのブロック内で完結させる
    {
        let first = &mut v[0];
        *first += 10;
    } // ここで first の借用が解放される

    v.push(4); // もう競合しない

    // 値だけ欲しいならコピーして借用を残さない
    let copied = v[0];
    v.push(5);
    println!("{copied} {:?}", v);
}
```

要点は3つです。

- **借用の生存期間を短くする**: ブロック `{ }` で囲い、可変借用を必要な箇所だけで完結させる
- **使用順を入れ替える**: `push` などの可変操作を先に済ませてから要素を借用する
- **値をコピーして持つ**: 参照ではなく `let copied = v[0];` のように値を取れば、借用自体が残らない

借用を「短く・重ねない」ことで、`cannot borrow as mutable more than once at a time` は解消できます。

---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/agentrules260610)- 原因を根本から理解するなら体系的な技術書（Amazon: [Rust 実践 入門](https://www.amazon.co.jp/s?k=Rust%20%E5%AE%9F%E8%B7%B5%20%E5%85%A5%E9%96%80&tag=1280itsuya22-22)）
