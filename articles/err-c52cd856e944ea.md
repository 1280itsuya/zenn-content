---
title: "error: failed to push some refs to"
emoji: "🐛"
type: "tech"
topics: ["Git", "GitHub", "エラー解決"]
published: true
---

## 発生条件

- ローカルブランチにコミットを積んだ後、`git push` を実行したとき、リモートリポジトリに自分が知らないコミットが存在している場合
- チームメンバーが先にリモートへプッシュしていて、手元のブランチがそれより古い状態（non-fast-forward）のとき
- `git fetch` や `git pull` をせずに複数の環境から同一ブランチへプッシュしようとしたとき

## 原因

`error: failed to push some refs to` は、ローカルの `HEAD` がリモートの最新コミットを親として持っていないときに発生する。Gitはデフォルトでfast-forwardマージしかプッシュを許可しないため、履歴が分岐していると拒否される。

最小再現手順：

```bash
# 環境A でコミットしてプッシュ
git commit -m "A's change"
git push origin main

# 環境B は fetch せずに別のコミットを積んでプッシュ → 失敗
git commit -m "B's change"
git push origin main
# error: failed to push some refs to 'https://github.com/user/repo.git'
# hint: Updates were rejected because the remote contains work that you do not have locally.
```

リモートにコミットCがあるのに、ローカルはCを知らない状態でプッシュを試みるため、`failed to push some refs to` が返る。

## 修正方法

リモートの変更を取り込んでからプッシュする。`--rebase` を使うと、自分のコミットをリモートの先頭に積み直すため、マージコミットが増えずに履歴が直線を保てる。

```bash
# リモートの変更を rebase で取り込む
git pull --rebase origin main

# コンフリクトが出た場合は解消してから
git add <修正したファイル>
git rebase --continue

# 問題なければプッシュ
git push origin main
```

`git pull --rebase` は内部的に `git fetch` → `git rebase` の順に実行する。`git pull`（デフォルトのマージ）でも `failed to push some refs to` は解消できるが、その場合はマージコミットが1つ増える。チームの運用方針に合わせて使い分けること。

なお、`--force` や `--force-with-lease` によるプッシュはリモートの変更を上書きするため、共有ブランチでは基本的に使わない。自分だけが使うfeatureブランチでrebaseした後に押し直す場合に限り、`--force-with-lease`（相手の変更がないことを確認してから上書き）を検討する。

```bash
# feature ブランチ限定・自分専用のケースのみ
git push --force-with-lease origin feature/my-branch
```

共有の `main` や `develop` ブランチに対してforceプッシュするとチームメンバーのローカル履歴と乖離するため、原則禁止とすること。

## 似たエラーの解決記事

- [fatal: not a git repository (or any of the parent directories): .git](https://zenn.dev/articles/err-332797515e19b4)


---

📌 このエラーの要点だけ知りたい方は [エラー即答ページ](https://1280itsuya.github.io/devtools/error/e/b1c666adf89e/) へ。別のエラーで困ったら [エラーメッセージ検索ツール](https://1280itsuya.github.io/devtools/error/)（貼るだけで原因と直し方を表示）も使えます。


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/aikit260702)- 原因を根本から理解するなら体系的な技術書（Amazon: [Git 実践 入門](https://www.amazon.co.jp/s?k=Git%20%E5%AE%9F%E8%B7%B5%20%E5%85%A5%E9%96%80&tag=1280itsuya22-22)）
