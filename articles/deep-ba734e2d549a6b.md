---
title: "【完全版】error[E0499]: cannot borrow as mutable more than once at a time"
emoji: "💎"
type: "tech"
topics: ["エラー解決"]
published: true
price: 300
---

## 何が起きているのか（原因の本質）

`error[E0499]: cannot borrow as mutable more than once at a time` は、Rustのコンパイラが「同一データへの可変借用が同時に2つ以上存在する」と判定したときに発生する。表面上は「2回借りようとしたから怒られた」に見えるが、内部で起きていることはもう少し深い。

Rustの借用規則は「ある瞬間に可変参照は1つだけ」を保証することで、データ競合をコンパイル時に完全に除去する。これはスレッド安全性だけでなく、同一スレッド内でも適用される。可変参照が2つ並ぶと、一方が保持するポインタの指す先をもう一方が書き換えられる。たとえば`Vec`への2つの`&mut`が同時に存在し、一方が`push`でバッファを再確保した場合、他方のポインタはダングリングになる。これを言語レベルで禁止しているのがこのエラーの本質だ。

重要なのは「ライフタイムの重なり」という概念だ。Rust 2018以降はNon-Lexical Lifetimes（NLL）が導入されており、借用のスコープはレキシカルなブロック境界ではなく「最後に使われた箇所まで」に短縮される。それでもE0499が出るケースが後を絶たない。なぜかというと、NLLが解決するのはあくまでスコープが見た目上分離しているケースに限られ、借用が構造体のフィールド越しに伝播するケースや、メソッド呼び出しが返す参照のライフタイムが`self`全体に結びついているケースでは、コンパイラは「重なっている」と判定し続けるからだ。

もう一点押さえておきたいのは、コンパイラが借用を追跡する単位だ。現行のNLL実装は基本的にフィールド単位まで借用を分割できるが（Partial Borrows）、メソッドの返り値の場合はそのメソッドのシグネチャが`&mut self`を取る時点で`self`全体が占有される。複数のフィールドへ個別にアクセスしたいのに、単一の`&mut self`メソッドが壁になって連鎖するケースはこれが原因だ。

---

## なぜ多くの解説では解決しないのか

ネット上にある説明の大半は「スコープを分けろ」「クローンしろ」で終わる。単純なケースではそれで動くが、実際に困るのはそれでは解決しない状況だ。

**パターン1：メソッドが`&mut self`を返す**

```rust
fn first_mut(&mut self) -> &mut i32 { &mut self.data[0] }
```

このようなメソッドを呼んで得た参照を保持しながら同じ構造体のメソッドを再度呼ぼうとすると、返り値の参照ライフタイムが`'self`に束縛されているため、コンパイラは「まだselfを借用中」と判断してE0499を出す。スコープを視覚的に分けても、返り値を変数に保持している限り借用は続いている。

**パターン2：HashMapのエントリ更新**

```rust
let val = map.get_mut("key").unwrap();
map.insert("other", compute(val)); // ← E0499
```

`get_mut`で得た`val`は`map`全体への`&mut`借用を保持する。その状態で`insert`を呼ぶのは2つ目の`&mut`になる。「同じキーじゃないから大丈夫では」と思うが、コンパイラはHashMapの内部実装を見ない。シグネチャだけで判断するため、`map`全体が占有されているとみなされる。

**パターン3：ループ内での借用の延命**

```rust
for i in 0..v.len() {
    let r = &mut v[i];
    v.push(0); // ← E0499
}
```

`r`がまだ有効なうちに`push`を呼ぶと二重借用になる。「ループの次のイテレーションでrを使わない」と人間には明らかでも、コンパイラは`r`の使用箇所を静的に追跡するため`push`の行の時点で「rはまだ生きている」と判断することがある。

これらはいずれも「スコープを分ける」だけでは解決しない。根本的に構造を変える必要がある。

---

## 完全な再現と修正コード

### 最小再現コード

```rust
fn main() {
    let mut v: Vec<i32> = vec![1, 2, 3];

    let first = &mut v[0]; // 1回目の可変借用
    let second = &mut v[1]; // E0499: cannot borrow `v` as mutable more than once at a time

    *first += 10;
    *second += 20;
    println!("{:?}", v);
}
```

上記は1回目の`first`が有効な間に2回目の`second`を取ろうとするためE0499が出る。

### 修正コード（インデックス経由でアクセス）

```rust
fn main() {
    let mut v: Vec<i32> = vec![1, 2, 3];

    // 参照を持つ代わりにインデックスで操作
    v[0] += 10;
    v[1] += 20;
    println!("{:?}", v);
}
```

参照を変数に保持しないことで借用が各ステートメントで完結し、重なりが生じない。

### split_at_mutを使って同時に複数要素を操作する場合

```rust
fn main() {
    let mut v: Vec<i32> = vec![1, 2, 3];

    let (left, right) = v.split_at_mut(1);
    let first = &mut left[0];
    let second = &mut right[0]; // 異なるスライスなので安全

    *first += 10;
    *second += 20;
    println!("{:?}", v);
}
```

`split_at_mut`は内部でunsafeを使い、重ならないことを保証したうえで2つの`&mut`スライスを返す。コンパイラはこれを安全と判断できる。

### HashMapのエントリ更新

```rust
use std::collections::HashMap;

fn main() {
    let mut map: HashMap<&str, i32> = HashMap::new();
    map.insert("a", 1);

    // NG: get_mutの参照を保持したままinsert
    // let val = map.get_mut("a").unwrap();
    // map.insert("b", *val + 1); // E0499

    // OK: entry APIを使う
    map.entry("b").or_insert_with(|| {
        let a = map["a"]; // ← これもE0499になる。下の方法を使う
        a + 1
    });
    // 正しい方法: 先に値を読み出してから挿入
    let a_val = map["a"];
    map.entry("b").or_insert(a_val + 1);

    println!("{:?}", map);
}
```

必要な値を`&`で先に読み出し、可変借用が終わってから`entry`を呼ぶのが定石だ。

### 構造体のフィールドを個別に借用する

```rust
struct State {
    data: Vec<i32>,
    cache: Vec<i32>,
}

impl State {
    fn update(&mut self) {
        // NG: selfを経由するメソッド2連鎖
        // let d = self.get_data_mut();
        // self.update_cache(d); // E0499

        // OK: フィールドに直接アクセスして分割借用
        let data = &mut self.data;
        let cache = &mut self.cache;
        for (i, d) in data.iter().enumerate() {
            if let Some(c) = cache.get_mut(i) {
                *c = *d * 2;
            }
        }
    }
}

fn main() {
    let mut s = State { data: vec![1, 2, 3], cache: vec![0, 0, 0] };
    s.update();
    println!("{:?}", s.cache);
}
```

フィールドに直接アクセスすれば、コンパイラは`self.data`と`self.cache`を独立した借用として扱える。

---

## 本番投入前チェックリスト

- [ ] 可変参照を変数に束縛するときに、その変数が本当に必要なだけの期間しか生きていないかを確認した（NLLが自動で短縮してくれることを期待しすぎない）
- [ ] `&mut self`を取るメソッドが返す参照を変数に保持したまま、同じ`self`の別メソッドを呼んでいないかを確認した
- [ ] `HashMap`や`BTreeMap`で`get_mut`の返り値を保持したままマップ全体を操作しようとしていないか確認し、必要であれば`entry` APIへ書き換えた
- [ ] ループ内でコレクションへの参照とそのコレクションへの変更操作が混在していないかを確認した（参照の代わりにインデックスを使う方針に切り替えた）
- [ ] 複数フィールドを同時に可変操作する必要がある場合は、メソッド越しでなくフィールドへの直接アクセスで分割借用できるか確認した
- [ ] それでも解決できない場合は`RefCell<T>`（単一スレッド）または`Mutex<T>`（複数スレッド）による内部可変性パターンを検討したうえで、実行時パニックのリスクを把握した
- [ ] `unsafe`を使って回避しようとしている場合、`split_at_mut`など標準ライブラリが提供する安全な代替手段を先にすべて検討した

---

## 関連する2つのエラーへの応用

### E0502: cannot borrow as mutable because it is also borrowed as immutable

```
error[E0502]: cannot borrow `x` as mutable because it is also borrowed as immutable
```

E0499が「可変 + 可変」の競合であるのに対し、E0502は「不変借用が生きている間に可変借用を取ろうとした」ケースだ。

```rust
let mut v = vec![1, 2, 3];
let r = &v[0];       // 不変借用
v.push(4);           // 可変借用: E0502
println!("{}", r);
```

本記事の考え方をそのまま転用できる。`r`が必要な値は先にコピーしてしまい、不変借用を手放してから可変操作を行う。

```rust
let mut v = vec![1, 2, 3];
let val = v[0];      // 値をコピー（借用ではなくムーブまたはコピー）
v.push(4);
println!("{}", val);
```

`Copy`トレイトを実装している型（`i32`など）はこれで解決する。参照そのものが必要な場合は、操作の順序を入れ替えて借用が重ならないようにする発想はE0499と同じだ。

### E0506: cannot assign to `x` because it is borrowed

```
error[E0506]: cannot assign to `x` because it is borrowed
```

変数に対する参照が生きている間に、その変数自体を上書き代入しようとするエラーだ。

```rust
let mut s = String::from("hello");
let r = &s;
s = String::from("world"); // E0506
println!("{}", r);
```

一見E0499と別物に見えるが、根底の論理は同じだ。「既存の借用が有効な間はデータの所有権操作を許さない」というルールの別の現れに過ぎない。`s`を上書きするということは古いヒープデータが解放される可能性があり、`r`がダングリング参照になる恐れがある。解法も本記事と同じ方向性で、「`r`を使い終わるまで上書きを遅らせる」か「先に`r`の値を取り出してから上書きする」かのどちらかだ。E0499で身につけた「借用の重なりを時系列で考える」視点をそのまま適用すればよい。

---

簡易版(無料)はこちら: [無料記事](https://zenn.dev/articles/err-e7d0176f3b29cf)
