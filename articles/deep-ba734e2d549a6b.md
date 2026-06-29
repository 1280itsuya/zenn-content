---
title: "【完全版】error[E0499]: cannot borrow as mutable more than once at a time"
emoji: "💎"
type: "tech"
topics: ["エラー解決"]
published: true
price: 300
---

## 何が起きているのか（原因の本質）

`error[E0499]: cannot borrow as mutable more than once at a time` は、Rust のコンパイラが**同一値への可変借用が同時に2つ以上存在する**と判断したときに発生する。

Rust の所有権システムは次の不変条件を強制する。

> ある値への可変参照（`&mut T`）が生きている間、同じ値への別の参照（`&T` も `&mut T` も）は存在できない。

この制約はデータ競合をコンパイル時にゼロにするための設計であり、ランタイムではなくコンパイラが静的に証明する。

問題の核心は**借用のスコープがソースコードの見た目と必ずしも一致しない**点にある。Rust 2018 edition 以降は NLL（Non-Lexical Lifetimes）が導入され、変数の最後の使用時点で借用が終了するよう改善された。しかし NLL でも解消できないパターンが存在する。代表例がコレクション要素への参照を保持しながら同一コレクションを変更しようとするケースだ。

```rust
let mut v = vec![1, 2, 3];
let first = &mut v[0]; // v への可変借用①が始まる
v.push(4);             // v への可変借用②→ E0499
println!("{}", first);
```

`v.push(4)` は内部で `&mut self` を要求するため、借用①がまだ生きている状態で借用②が発生する。コンパイラは `first` が最後に使われる `println!` まで借用①を有効とみなすため、`push` の行でエラーになる。

借用の「生存区間」はあくまでコンパイラが計算するライフタイムグラフ上の話であり、人間が直感するスコープとズレが生じやすい。このズレがエラーを追いにくくさせている。

---

## なぜ多くの解説では解決しないのか

### 「スコープを絞ればいい」という助言が効かないケース

ブロックで囲んで借用を早期に終わらせる方法はよく紹介される。しかし**参照を返すメソッド（`entry` API など）を介して借用が継続しているケース**では、ブロックを置いた位置と借用終端がかみ合わないことがある。

```rust
use std::collections::HashMap;

fn add_or_increment(map: &mut HashMap<String, u32>, key: &str) {
    let entry = map.entry(key.to_string()).or_insert(0); // 可変借用①
    *entry += 1;
    // entry はまだここで生きている
    map.remove("other_key"); // 可変借用②→ E0499
}
```

`entry` が生きている限り `map` への別の可変操作は通らない。「スコープを絞った」つもりでも `entry` の使用が後続行にある限りエラーは消えない。

### `clone()` で回避したが意図と違う結果になるケース

`clone()` でデータをコピーすれば借用の競合は消えるが、クローン後に元データを変更しても参照先には反映されない。副作用を期待しているコードで静かにバグになる。

### イテレータ中の変更

コレクションをイテレートしながら同一コレクションに push や remove をしようとすると E0499 が出る。Java や Python では実行時例外になる操作をコンパイル時に弾く設計のため、「他言語では動いた」感覚のある開発者が最初に詰まるパターンだ。

---

## 完全な再現と修正コード

### 最小再現コード（コンパイルエラーになる）

```rust
fn main() {
    let mut numbers = vec![10, 20, 30];

    // 要素への可変参照を取得
    let first = &mut numbers[0];

    // 参照が生きている間にコレクション自体を変更しようとする
    numbers.push(40); // error[E0499]

    println!("first = {}", first);
}
```

コンパイルすると以下のメッセージが出る。

```
error[E0499]: cannot borrow `numbers` as mutable more than once at a time
 --> src/main.rs:7:5
  |
4 |     let first = &mut numbers[0];
  |                      ------- first mutable borrow occurs here
7 |     numbers.push(40);
  |     ^^^^^^^ second mutable borrow occurs here
9 |     println!("first = {}", first);
  |                            ----- first borrow later used here
```

### 修正パターン①：先に操作を終わらせる

```rust
fn main() {
    let mut numbers = vec![10, 20, 30];

    // push を先に済ませてから要素を参照する
    numbers.push(40);

    let first = &mut numbers[0];
    *first *= 2;

    println!("numbers = {:?}", numbers); // [20, 20, 30, 40]
}
```

参照を取る前に `push` を終わらせることで借用の競合がなくなる。

### 修正パターン②：インデックスでアクセスし参照を持ち越さない

```rust
fn main() {
    let mut numbers = vec![10, 20, 30];

    // 参照を変数に束縛せず、インデックスアクセスで完結させる
    numbers[0] *= 2;
    numbers.push(40);

    println!("numbers = {:?}", numbers); // [20, 20, 30, 40]
}
```

### 修正パターン③：`HashMap` の `entry` 競合を回避

```rust
use std::collections::HashMap;

fn add_or_increment(map: &mut HashMap<String, u32>, key: &str) {
    // entry の借用スコープをブロックで明示的に終わらせる
    {
        let entry = map.entry(key.to_string()).or_insert(0);
        *entry += 1;
    } // ここで entry が drop され、map への借用が解放される

    map.remove("other_key"); // これで通る
}

fn main() {
    let mut m = HashMap::new();
    m.insert("other_key".to_string(), 99u32);
    add_or_increment(&mut m, "target");
    println!("{:?}", m);
}
```

ブロックが有効なのは、`entry` が `drop` されることで借用①が確実に終端するためだ。NLL だけに頼らず明示的にスコープを切るのが確実な手法になる。

---

## 本番投入前チェックリスト

- [ ] コレクション要素への `&mut` 参照を変数に束縛した後、同一コレクションへの操作（`push` / `remove` / `insert` / `sort` など）を行っていないか確認する
- [ ] `HashMap::entry()` / `BTreeMap::entry()` を使った後、同一マップを変更する処理がエントリ参照のスコープ外にあるかブロックで明示的に囲んであるか確認する
- [ ] イテレータ（`iter_mut()` を含む）の生存中に同一コレクションへの変更操作が存在しないか確認する（必要なら `indices` を先に収集してから別ループで変更する）
- [ ] `clone()` で回避した箇所が「本当にコピーで十分か」を設計意図と照合する（参照先への副作用が必要な箇所で誤用するとサイレントバグになる）
- [ ] `RefCell<T>` / `Mutex<T>` を使って借用チェックを実行時に先送りした場合、`borrow_mut()` のパニック条件（複数の可変借用が同時に取られるケース）をテストで網羅しているか確認する

---

## 関連する2つのエラーへの応用

### error[E0502]: cannot borrow as immutable because it is also borrowed as mutable

```
error[E0502]: cannot borrow `x` as immutable because it is also borrowed as mutable
```

E0499 が「可変 vs 可変」の競合であるのに対し、E0502 は「可変借用が生きている間に不変借用を取ろうとした」場合に発生する。

```rust
let mut v = vec![1, 2, 3];
let m = &mut v;       // 可変借用
let r = &v;           // E0502: 不変借用
println!("{:?}", m);
```

本記事と同じ解決アプローチが有効だ。**借用の生存区間を重複させないよう操作順を整理するか、明示的なブロックで可変借用を終端させる**。`m` を使い終えてから `r` を取るよう順番を変えれば解消する。

### error[E0506]: cannot move out of because it is borrowed

```
error[E0506]: cannot move out of `x` because it is borrowed
```

借用が生きている間に値の所有権を move しようとしたときに発生する。E0499 の「借用同士の競合」とは少し異なるが、根本原因は同じ「借用のライフタイムが重複している」点だ。

```rust
let mut s = String::from("hello");
let r = &s;
let owned = s; // E0506: r が生きている間に move
println!("{}", r);
```

解決策も共通している。**参照 `r` の最後の使用を move より前に済ませるか、`clone()` で所有権を分離する**。NLL が参照の最終使用点を正しく検出できる単純なケースではコード順の調整だけで通ることが多い。複雑な制御フローでは明示的なブロックで借用を終端させる本記事のパターンがそのまま転用できる。

---

簡易版(無料)はこちら: [無料記事](https://zenn.dev/articles/err-e7d0176f3b29cf)
