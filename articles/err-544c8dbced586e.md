---
title: "Fatal error: Uncaught Error: Call to undefined function"
emoji: "🐛"
type: "tech"
topics: ["PHP", "Laravel", "エラー解決"]
published: true
---

## 発生条件

- PHPファイル内で呼び出した関数が、そのスクリプト内にも読み込まれたファイル内にも定義されていない場合
- `mysqli_connect()` や `mb_strlen()` など、PHP拡張モジュールが必要な組み込み関数を、対応する拡張が無効な状態で呼び出した場合
- Composerで管理しているライブラリの関数を、`vendor/autoload.php` を読み込まずに使おうとした場合

## 原因

`Fatal error: Uncaught Error: Call to undefined function` は、PHPがその関数の定義を見つけられないときに投げるエラーです。

最もよくあるパターンを三つ示します。

**パターン1: 関数名のスペルミス**

```php
<?php
function greet(string $name): string {
    return "Hello, " . $name;
}

// 関数名を間違えて呼び出す
echo grtte("World"); // Fatal error: Call to undefined function grtte()
```

**パターン2: 拡張モジュールが無効**

```php
<?php
$str = "こんにちは";

// mbstring拡張が無効だとここで落ちる
echo mb_strlen($str); // Fatal error: Call to undefined function mb_strlen()
```

**パターン3: autoloadの読み込み忘れ**

```php
<?php
// vendor/autoload.php を require していない

use GuzzleHttp\Client;

$client = new Client();
// あるいはライブラリ提供のヘルパー関数を直接呼ぶと同様に落ちる
```

## 修正方法

**パターン1: スペルを修正する**

```php
<?php
function greet(string $name): string {
    return "Hello, " . $name;
}

echo greet("World"); // 正しい関数名に修正
```

関数名は大文字・小文字を区別しませんが、スペルミスには区別がありません。IDEの補完を活用するか、`function_exists()` で事前確認するのが確実です。

```php
<?php
if (function_exists('greet')) {
    echo greet("World");
} else {
    // 関数が存在しない場合の処理
    throw new RuntimeException("関数 greet が見つかりません");
}
```

**パターン2: php.ini で拡張を有効化する**

`php.ini` を開き、該当行のコメントアウトを外します。

```ini
; 変更前
;extension=mbstring

; 変更後
extension=mbstring
```

変更後はWebサーバー(ApacheやNginx+PHP-FPM)を再起動してください。有効になっているかどうかは `phpinfo()` またはCLIで確認できます。

```bash
php -m | grep mbstring
```

`mbstring` と表示されれば有効です。

**パターン3: autoloadを読み込む**

```php
<?php
// スクリプトの先頭でautoloadを読み込む
require_once __DIR__ . '/vendor/autoload.php';

use GuzzleHttp\Client;

$client = new Client();
```

ライブラリ自体がインストールされていない場合は、先にComposerで追加します。

```bash
composer require guzzlehttp/guzzle
```

`composer.json` に依存関係が追記され、`vendor/` 以下にファイルが展開されます。その後、上記のように `vendor/autoload.php` を読み込めば `Call to undefined function` は解消されます。

---

📌 このエラーの要点だけ知りたい方は [エラー即答ページ](https://1280itsuya.github.io/devtools/error/e/9b3c1228cee0/) へ。別のエラーで困ったら [エラーメッセージ検索ツール](https://1280itsuya.github.io/devtools/error/)（貼るだけで原因と直し方を表示）も使えます。


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/aikit260626)- 原因を根本から理解するなら体系的な技術書（Amazon: [PHP Laravel 実践](https://www.amazon.co.jp/s?k=PHP%20Laravel%20%E5%AE%9F%E8%B7%B5&tag=1280itsuya22-22)）
