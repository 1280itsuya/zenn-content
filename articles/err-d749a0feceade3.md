---
title: "Error: connect ECONNREFUSED 127.0.0.1:5432"
emoji: "🐛"
type: "tech"
topics: ["Node.js", "JavaScript", "エラー解決", "npm"]
published: true
---

## 発生条件

- Node.js や [Python](https://www.amazon.co.jp/s?k=Python%20%E5%85%A5%E9%96%80%20%E6%9C%AC&tag=1280itsuya22-22) などのアプリケーションが PostgreSQL に接続しようとしたとき、PostgreSQL サーバーがそもそも起動していない場合
- 接続先の host または port が実際のサーバーと食い違っている場合（例: ポートを `5432` と指定しているが実際は `5433` で動いている）
- Docker Compose などのコンテナ環境で、DB コンテナが起動し終わる前にアプリが接続を試みた場合

## 原因

`ECONNREFUSED 127.0.0.1:5432` は「指定した IP とポートにそもそも誰も Listen していない」というカーネルレベルの拒否を意味します。接続タイムアウトではなく即座に返るのが特徴です。

以下は Node.js (`pg` ライブラリ) での最小再現例です。PostgreSQL が停止している、またはポートが違う状態で実行すると `ECONNREFUSED` が発生します。

```javascript
const { Client } = require('pg');

const client = new Client({
  host: '127.0.0.1',
  port: 5432,      // ← 実際にListenしているポートと違う場合
  database: 'mydb',
  user: 'myuser',
  password: 'mypassword',
});

client.connect(); // Error: connect ECONNREFUSED 127.0.0.1:5432
```

Python (`psycopg2`) でも同様です。

```python
import psycopg2

conn = psycopg2.connect(
    host="127.0.0.1",
    port=5432,       # ← サーバーが停止/別ポートならここで拒否される
    dbname="mydb",
    user="myuser",
    password="mypassword",
)
```

## 修正方法

**Step 1: PostgreSQL が起動しているか確認する**

```bash
# Linux / macOS (systemd)
sudo systemctl status postgresql

# 起動していなければ
sudo systemctl start postgresql

# macOS (Homebrew)
brew services start postgresql@15

# Docker の場合
docker ps | grep postgres
docker start <コンテナ名>
```

**Step 2: 実際に Listen しているポートを確認する**

```bash
# Linux / macOS
sudo ss -tlnp | grep postgres
# または
sudo lsof -i :5432

# PostgreSQL が別ポートで動いているなら出力に表示される
# 例: *:5433 で Listen していれば port を 5433 に合わせる
```

**Step 3: 接続設定を実際の値に合わせる**

```javascript
// Node.js (pg) の修正例
const client = new Client({
  host: process.env.DB_HOST || '127.0.0.1',
  port: parseInt(process.env.DB_PORT || '5432'),
  database: process.env.DB_NAME || 'mydb',
  user: process.env.DB_USER || 'myuser',
  password: process.env.DB_PASSWORD || 'mypassword',
});
```

```python
# Python (psycopg2) の修正例
import os
import psycopg2

conn = psycopg2.connect(
    host=os.getenv("DB_HOST", "127.0.0.1"),
    port=int(os.getenv("DB_PORT", 5432)),
    dbname=os.getenv("DB_NAME", "mydb"),
    user=os.getenv("DB_USER", "myuser"),
    password=os.getenv("DB_PASSWORD", "mypassword"),
)
```

環境変数経由にしておくと、ローカル・Docker・本番で host/port を切り替えるだけで済みます。

**Docker Compose の場合の注意点**

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:15
    ports:
      - "5432:5432"

  app:
    depends_on:
      db:
        condition: service_healthy  # ← healthcheck が通るまで待つ
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser"]
      interval: 5s
      retries: 5
```

`depends_on` だけでは DB の起動完了を保証できないため、`condition: service_healthy` と `healthcheck` をセットで使います。これにより `ECONNREFUSED 127.0.0.1:5432` が起動順序の競合から発生するケースを防げます。

## 似たエラーの解決記事

- [SyntaxError: Cannot use import statement outside a module](https://zenn.dev/articles/err-85d63d44f92d4a)
- [Error: listen EADDRINUSE: address already in use :::3000](https://zenn.dev/articles/err-3ae8d1b64d57da)
- [FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory](https://zenn.dev/articles/err-de059f5f9860b4)


---

📌 このエラーの要点だけ知りたい方は [エラー即答ページ](https://1280itsuya.github.io/devtools/error/e/f5605db8f27e/) へ。別のエラーで困ったら [エラーメッセージ検索ツール](https://1280itsuya.github.io/devtools/error/)（貼るだけで原因と直し方を表示）も使えます。


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/aikit260629)- 原因を根本から理解するなら体系的な技術書（Amazon: [Node.js 実践入門](https://www.amazon.co.jp/s?k=Node.js%20%E5%AE%9F%E8%B7%B5%E5%85%A5%E9%96%80&tag=1280itsuya22-22)）
