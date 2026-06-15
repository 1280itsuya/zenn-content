---
title: "Error response from daemon: Conflict. The container name is already in"
emoji: "🐛"
type: "tech"
topics: ["Docker", "コンテナ", "エラー解決"]
published: true
---

## 発生条件

- `docker run --name <コンテナ名>` を実行した際、同じ名前のコンテナがすでに存在している
- 前回のコンテナが `Exited` 状態のまま残っており、`docker ps` では見えないが `docker ps -a` には表示される
- `docker-compose up` を `down` せずに再実行したとき、内部で割り当てようとした名前が衝突する

## 原因

Dockerはコンテナ名をグローバルに一意に管理する。停止中（`Exited`）のコンテナも名前を占有し続けるため、同名で新たに起動しようとすると次のエラーが出る。

```
Error response from daemon: Conflict. The container name "/my-app" is already in use by container "3f8a2b..."
```

最小再現手順は以下のとおり。

```bash
# 1回目は成功する
docker run --name my-app nginx

# コンテナを stop するが rm はしない
docker stop my-app

# 2回目: 同名で再実行 → Conflict エラーが発生
docker run --name my-app nginx
```

`docker ps` の出力にはすでに停止したコンテナは表示されないが、`docker ps -a` を実行すると `STATUS` が `Exited` のまま `my-app` が残っていることが確認できる。これが `Conflict. The container name is already in use` の直接原因となる。

## 修正方法

**方法1: 既存コンテナを削除してから再実行する**

```bash
# 停止中のコンテナを削除
docker rm my-app

# 改めて起動
docker run --name my-app nginx
```

コンテナが起動中の場合は先に停止が必要。

```bash
docker stop my-app && docker rm my-app
docker run --name my-app nginx
```

**方法2: `--rm` フラグを使い、停止時に自動削除させる**

```bash
docker run --rm --name my-app nginx
```

コンテナが停止するたびに自動削除されるため、次回実行時に名前の衝突が起きない。ただしコンテナ停止後にログを `docker logs` で参照したい場合は使えない点に注意する。

**方法3: `--name` を変えて別名で起動する**

```bash
docker run --name my-app-v2 nginx
```

既存コンテナを残したまま新しい名前で起動する。古いコンテナが不要になった時点で `docker rm my-app` で削除する。

**Exited コンテナをまとめて掃除したい場合**

```bash
# 停止中の全コンテナを一括削除
docker container prune
```

確認プロンプトが出るので、削除して問題ないか事前に `docker ps -a` で確認してから実行する。

---

📌 このエラーの要点だけ知りたい方は [エラー即答ページ](https://1280itsuya.github.io/devtools/error/e/2680f58e278b/) へ。別のエラーで困ったら [エラーメッセージ検索ツール](https://1280itsuya.github.io/devtools/error/)（貼るだけで原因と直し方を表示）も使えます。


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/aikit260615)- 原因を根本から理解するなら体系的な技術書（Amazon: [Docker 実践 入門](https://www.amazon.co.jp/s?k=Docker%20%E5%AE%9F%E8%B7%B5%20%E5%85%A5%E9%96%80&tag=1280itsuya22-22)）
