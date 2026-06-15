---
title: "Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is"
emoji: "🐛"
type: "tech"
topics: ["Docker", "コンテナ", "エラー解決"]
published: true
---

## 発生条件

- `docker ps` や `docker run` など任意の Docker コマンドを実行したとき、Docker Desktop（または dockerd）がまだ起動していない場合
- Linux サーバーで `systemctl start docker` を実行していない状態でスクリプトやCI ジョブが走った場合
- Docker Desktop をインストール直後、PC 再起動後、またはサービスがクラッシュした直後に操作しようとした場合

## 原因

Docker CLI はコマンドをソケットファイル `unix:///var/run/docker.sock` 経由で Docker daemon（`dockerd`）に送信する。daemon が停止していると、このソケット自体が存在しないためエラーになる。

```bash
# daemon が停止している状態で実行すると再現する
$ docker ps
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

エラーメッセージ末尾の `Is the docker daemon running?` が示す通り、CLI 側の問題ではなく daemon 側が起動していないことが直接の原因である。ソケットファイルの有無を確認するだけでも切り分けができる。

```bash
# ソケットが存在しなければ daemon は停止している
ls -la /var/run/docker.sock
# ls: cannot access '/var/run/docker.sock': No such file or directory
```

## 修正方法

**macOS / Windows（Docker Desktop）**

Docker Desktop アプリを起動する。GUI から起動するか、コマンドラインで以下を実行する。

```bash
# macOS: open コマンドで Docker Desktop を起動
open -a Docker

# 起動完了を待ってから確認
docker info
```

**Linux（systemd 環境）**

```bash
# daemon を起動
sudo systemctl start docker

# OS 再起動時に自動起動させる場合
sudo systemctl enable docker

# 状態確認
sudo systemctl status docker

# 接続確認
docker ps
```

**Linux（systemd 以外 / 手動起動）**

```bash
sudo dockerd &
```

起動後に再度 `docker ps` を実行し、コンテナ一覧が返れば `unix:///var/run/docker.sock` への接続が回復している。CI/CD 環境では、ジョブ実行前に `systemctl is-active docker` でサービス状態をチェックするステップを挟むと、同様のエラーを事前に検知できる。

```yaml
# GitHub Actions での例（self-hosted runner）
- name: Check Docker daemon
  run: |
    sudo systemctl is-active docker || sudo systemctl start docker
    docker info

---

📌 このエラーの要点だけ知りたい方は [エラー即答ページ](https://1280itsuya.github.io/devtools/error/e/f8af3384c57c/) へ。別のエラーで困ったら [エラーメッセージ検索ツール](https://1280itsuya.github.io/devtools/error/)（貼るだけで原因と直し方を表示）も使えます。


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/aikit260615)- 原因を根本から理解するなら体系的な技術書（Amazon: [Docker 実践 入門](https://www.amazon.co.jp/s?k=Docker%20%E5%AE%9F%E8%B7%B5%20%E5%85%A5%E9%96%80&tag=1280itsuya22-22)）
