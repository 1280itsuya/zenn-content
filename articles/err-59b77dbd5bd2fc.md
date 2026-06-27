---
title: "Permission denied"
emoji: "🐛"
type: "tech"
topics: ["シェルスクリプト", "Linux", "bash"]
published: true
---

## 発生条件

- シェルスクリプトやバイナリに実行権限（`x` ビット）が付与されていない状態で直接実行しようとしたとき
- 別ユーザー所有のファイルやディレクトリに対して書き込み・読み取りを試みたとき
- `sudo` なしで `/etc/` や `/usr/` 以下など、root 権限が必要なパスを操作しようとしたとき

## 原因

`Permission denied` は OS がプロセスのパーミッションチェックを行い、要求された操作が許可されていないと判断したときに返す EACCES エラーです。

最もよくある再現パターンは、ダウンロードしたスクリプトをそのまま実行しようとするケースです。

```bash
# スクリプトをダウンロードした直後の状態
$ ls -l script.sh
-rw-r--r-- 1 user user 128 Jun 27 09:00 script.sh

# 実行しようとすると Permission denied が発生する
$ ./script.sh
bash: ./script.sh: Permission denied
```

`-rw-r--r--` という表示が示すように、所有者・グループ・その他のいずれにも `x`（実行）ビットが立っていないため、シェルがファイルをプロセスとして起動できません。

[Python](https://www.amazon.co.jp/s?k=Python%20%E5%85%A5%E9%96%80%20%E6%9C%AC&tag=1280itsuya22-22) や Node.js で `open()` 系のファイル操作をする場合も同様に発生します。

```python
# 書き込み権限のないファイルへの書き込み
with open('/etc/hosts', 'w') as f:
    f.write('127.0.0.1 example.local')
# PermissionError: [Errno 13] Permission denied: '/etc/hosts'
```

`[Errno 13]` が `Permission denied` に対応する errno の値です。

## 修正方法

### シェルスクリプトの場合：実行権限を付与する

```bash
# 所有者に実行権限を追加
chmod +x script.sh

# 付与後の確認
ls -l script.sh
# -rwxr-xr-x 1 user user 128 Jun 27 09:00 script.sh

# 実行できるようになる
./script.sh
```

`chmod +x` は所有者・グループ・その他すべてに `x` を付与します。自分だけに限定したい場合は `chmod u+x script.sh` を使います。

### ファイルの所有者・権限を確認してから対処する

```bash
# 所有者と権限を確認
ls -la /path/to/file

# 自分のユーザー名を確認
whoami

# 所有者が root などで変更できない場合は sudo を使う
sudo chmod +x /path/to/script.sh

# 所有者ごと自分に変更したい場合
sudo chown $(whoami):$(whoami) /path/to/file
```

`sudo` は必要最小限の操作にだけ使い、`sudo chmod 777` のような過剰な権限付与は避けてください。第三者への読み書き実行を全開放することになり、共有サーバーでは特にセキュリティリスクになります。

### Python でファイル操作する場合：書き込み先を変える

```python
import os

target = '/etc/hosts'

# 書き込み前に権限を確認する
if not os.access(target, os.W_OK):
    print(f"書き込み権限がありません: {target}")
    # 書き込み可能なパスへ変更するか、sudo 付きで起動し直す
else:
    with open(target, 'a') as f:
        f.write('127.0.0.1 example.local\n')
```

`os.access()` で事前チェックを入れると、`Permission denied` を例外として落とすより手前で問題を検出できます。システムファイルを編集する必要があるスクリプトは、最初から `sudo python script.py` で起動するか、専用の設定管理ツール（Ansible 等）経由で操作するのが無難です。

## 似たエラーの解決記事

- [command not found](https://zenn.dev/articles/err-904434f799a130)


---

📌 このエラーの要点だけ知りたい方は [エラー即答ページ](https://1280itsuya.github.io/devtools/error/e/c52a14545185/) へ。別のエラーで困ったら [エラーメッセージ検索ツール](https://1280itsuya.github.io/devtools/error/)（貼るだけで原因と直し方を表示）も使えます。


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/aikit260627)- 原因を根本から理解するなら体系的な技術書（Amazon: [Linux シェル 実践](https://www.amazon.co.jp/s?k=Linux%20%E3%82%B7%E3%82%A7%E3%83%AB%20%E5%AE%9F%E8%B7%B5&tag=1280itsuya22-22)）
