---
title: "java.lang.ArrayIndexOutOfBoundsException"
emoji: "🐛"
type: "tech"
topics: ["Java", "エラー解決", "初心者"]
published: true
---

## 発生条件

`java.lang.ArrayIndexOutOfBoundsException` は以下の状況で発生する。

- 配列の長さが `n` のとき、`array[n]` や `array[-1]` のように存在しないインデックスへアクセスしたとき
- `for` ループの終了条件に `<=` を使い、`array[i]` が最後の要素を超えて参照されたとき
- 外部入力やメソッドの戻り値をそのままインデックスとして使い、範囲チェックを省略したとき

## 原因

典型的なパターンは、ループ条件に `length` をそのまま使ってしまうケースだ。

```java
public class Main {
    public static void main(String[] args) {
        int[] scores = {80, 90, 75};

        // 誤り: i <= scores.length のとき i=3 で scores[3] にアクセスしてしまう
        for (int i = 0; i <= scores.length; i++) {
            System.out.println(scores[i]); // i=3 で ArrayIndexOutOfBoundsException
        }
    }
}
```

`scores.length` は `3` だが、有効なインデックスは `0`・`1`・`2` の3つのみだ。`i <= scores.length` という条件は `i` が `3` になった時点でループを継続するため、存在しない `scores[3]` への参照が発生する。スタックトレースには `java.lang.ArrayIndexOutOfBoundsException: Index 3 out of bounds for length 3` のようにインデックス値と配列長が明示されるので、まずそこを確認する。

## 修正方法

```java
public class Main {
    public static void main(String[] args) {
        int[] scores = {80, 90, 75};

        // 修正: i < scores.length に変更する
        for (int i = 0; i < scores.length; i++) {
            System.out.println(scores[i]);
        }

        // または拡張 for 文を使えばインデックス管理が不要になる
        for (int score : scores) {
            System.out.println(score);
        }

        // 外部入力をインデックスとして使う場合は範囲チェックを必ず挟む
        int index = getUserInput(); // 外から来る値
        if (index >= 0 && index < scores.length) {
            System.out.println(scores[index]);
        } else {
            System.out.println("インデックスが範囲外です: " + index);
        }
    }

    static int getUserInput() {
        return 5; // 例として範囲外の値を返している
    }
}
```

修正の要点は3つある。

1. **ループ条件は `i < array.length`** を使う。`<=` は1つ多くループするため `java.lang.ArrayIndexOutOfBoundsException` の典型的な原因になる。
2. **インデックスを手動管理したくなければ拡張 `for` 文を使う**。要素を順に参照するだけなら添字が不要なため、境界違反そのものが起きない。
3. **外部由来の値をインデックスにするときは必ず `0 <= index < length` の範囲チェックを行う**。ユーザー入力やAPIレスポンス由来の数値は想定外の値を含む可能性があり、チェックなしで配列に渡すと本番環境で例外が発生する。

## 似たエラーの解決記事

- [Exception in thread "main" java.lang.NullPointerException](https://zenn.dev/articles/err-cc82dc0575f936)
- [java.lang.ClassNotFoundException](https://zenn.dev/articles/err-26339ac7d502ba)


---

📌 このエラーの要点だけ知りたい方は [エラー即答ページ](https://1280itsuya.github.io/devtools/error/e/d116323928a2/) へ。別のエラーで困ったら [エラーメッセージ検索ツール](https://1280itsuya.github.io/devtools/error/)（貼るだけで原因と直し方を表示）も使えます。


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/aikit260625)- 原因を根本から理解するなら体系的な技術書（Amazon: [Java 実践 入門](https://www.amazon.co.jp/s?k=Java%20%E5%AE%9F%E8%B7%B5%20%E5%85%A5%E9%96%80&tag=1280itsuya22-22)）
