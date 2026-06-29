---
title: "Error: listen EADDRINUSE: address already in use :::3000"
emoji: "🐛"
type: "tech"
topics: ["Node.js", "JavaScript", "エラー解決", "npm"]
published: true
---

## 発生条件

- Node.js や Next.js などのサーバーを起動しようとしたとき、既に同じポート番号でプロセスが動いている
- 開発サーバーを `Ctrl+C` で終了せずにターミナルを閉じた後、再度起動しようとした
- Docker コンテナやシステムサービスがポート 3000 を先に確保している状態でアプリを起動した

## 原因

`Error: listen EADDRINUSE: address already in use :::3000` は、Node.js が指定ポートをバインドしようとした際に、そのポートが既に別のプロセスに占有されていると発生する。

以下のような最小構成でも再現する。

```js
// server.js
const http = require('http');

const server1 = http.createServer((req, res) => res.end('first'));
server1.listen(3000);

// 同じポートで二つ目を起動しようとする
const server2 = http.createServer((req, res) => res.end('second'));
server2.listen(3000); // ここで EADDRINUSE が投げられる
```

`server2.listen(3000)` の時点で OS レベルのポートバインドに失敗し、`events.js` 内でハンドルされていない `error` イベントとしてプロセスがクラッシュする。

## 修正方法

### 方法 1: 占有プロセスを特定して終了する（macOS / Linux）

```bash
# ポート 3000 を使っている PID を調べる
lsof -i:3000

# 出力例:
# COMMAND   PID   USER   FD   TYPE  ...
# node     1234  user   23u  IPv6  ...

# PID を指定して終了
kill -9 1234
```

Windows の場合は PowerShell で以下を実行する。

```powershell
# ポート 3000 を使っているプロセスを探す
netstat -ano | findstr :3000

# PID が 1234 の場合
taskkill /PID 1234 /F
```

### 方法 2: ポート番号を変更する

起動時に別のポートを指定することで競合を回避できる。

```bash
# Next.js の場合
next dev -p 3001

# Create [React](https://www.amazon.co.jp/s?k=React%20%E5%85%A5%E9%96%80%20%E6%9C%AC&tag=1280itsuya22-22) App の場合
PORT=3001 npm start
```

環境変数ファイルで固定する場合は `.env.local` に記述する。

```
PORT=3001
```

### 方法 3: エラーを補足してポートを自動変更する

```js
const http = require('http');

const server = http.createServer((req, res) => res.end('ok'));

server.on('error', (err) => {
  if (err.code === 'EADDRINUSE') {
    console.error(`ポートが使用中です: ${err.message}`);
    // フォールバックポートで再試行
    server.listen(0); // 0 を指定するとOS が空きポートを自動割り当て
  }
});

server.listen(3000, () => {
  const addr = server.address();
  console.log(`listening on port ${addr.port}`);
});
```

`listen(0)` を使うと OS が空きポートを自動で割り当てるため、開発時の一時的な回避策として使える。本番では明示的なポート番号を指定することが望ましい。

## 似たエラーの解決記事

- [Error: Cannot find module 'express'](https://zenn.dev/articles/err-7b4f33b26a2d46)
- [Error [ERR_REQUIRE_ESM]: require() of ES Module not supported](https://zenn.dev/articles/err-6a5af89c21ca7b)
- [SyntaxError: Cannot use import statement outside a module](https://zenn.dev/articles/err-85d63d44f92d4a)


---

📌 このエラーの要点だけ知りたい方は [エラー即答ページ](https://1280itsuya.github.io/devtools/error/e/504aabf20015/) へ。別のエラーで困ったら [エラーメッセージ検索ツール](https://1280itsuya.github.io/devtools/error/)（貼るだけで原因と直し方を表示）も使えます。


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/aikit260629)- 原因を根本から理解するなら体系的な技術書（Amazon: [Node.js 実践入門](https://www.amazon.co.jp/s?k=Node.js%20%E5%AE%9F%E8%B7%B5%E5%85%A5%E9%96%80&tag=1280itsuya22-22)）
