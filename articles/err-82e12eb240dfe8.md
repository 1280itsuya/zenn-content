---
title: "Error: Inconsistent dependency lock file"
emoji: "🐛"
type: "tech"
topics: ["Terraform", "インフラ", "IaC"]
published: true
---

## 発生条件

- `.terraform.lock.hcl` がリポジトリに存在する状態で、`required_providers` のバージョン制約や対象プラットフォームを変更した後に `terraform init` を実行したとき
- チームメンバーが異なる OS（例: macOS と Linux）でそれぞれ `terraform init` を実行し、異なるプラットフォーム向けのハッシュが `.terraform.lock.hcl` に混在したとき
- プロバイダーのバージョンを手動で `terraform.lock.hcl` 外から書き換えた、あるいはファイル自体を削除せずに `required_providers` ブロックだけ更新したとき

## 原因

`Error: Inconsistent dependency lock file` は、`.terraform.lock.hcl` に記録されているプロバイダーのバージョンとハッシュが、現在の `required_providers` の制約と一致しないときに発生する。

以下は最小再現例として、`hashicorp/aws` のバージョン制約を変更したケースを示す。

```hcl
# main.tf（変更後）
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.50"   # ← ロックファイルには 5.0.0 系が記録されている
    }
  }
}
```

この状態でそのまま `terraform init` を実行すると、ロックファイルに書かれた古いバージョンのハッシュと `~> 5.50` という制約が矛盾するため、Terraform は処理を中断してエラーを返す。

```
Error: Inconsistent dependency lock file

The following dependency selections recorded in the lock file are inconsistent
with the current configuration:
  - provider registry.terraform.io/hashicorp/aws: locked version selection 5.0.1
    doesn't match the updated version constraint "~> 5.50.0"
```

## 修正方法

`-upgrade` フラグを付けて `terraform init` を再実行することで、`.terraform.lock.hcl` を現在の `required_providers` の制約に合わせて更新できる。

```bash
terraform init -upgrade
```

このコマンドは制約を満たす最新バージョンを解決し、ロックファイルのバージョンとハッシュを上書きする。その後、変更された `.terraform.lock.hcl` をリポジトリにコミットすることで、チーム全員が同じプロバイダーバージョンを使うことができる。

```bash
# ロックファイルを更新後にコミット
git add .terraform.lock.hcl
git commit -m "chore: update terraform lock file for aws provider ~> 5.50"
```

チームで複数 OS を混在させている場合は、`-platform` フラグで必要なプラットフォームのハッシュをロックファイルに追加しておくとよい。

```bash
terraform providers lock \
  -platform=linux_amd64 \
  -platform=darwin_arm64 \
  registry.terraform.io/hashicorp/aws
```

`Inconsistent dependency lock file` はロックファイルと設定のズレが原因であるため、ファイルを削除して再生成するのではなく、`-upgrade` や `providers lock` で正しく更新する方法を選ぶことが運用上安全である。

## 似たエラーの解決記事

- [Error: Reference to undeclared resource](https://zenn.dev/articles/err-e0ca0c08eb20e6)
- [Error: Provider configuration not present](https://zenn.dev/articles/err-3325038cea363a)


---

📌 このエラーの要点だけ知りたい方は [エラー即答ページ](https://1280itsuya.github.io/devtools/error/e/bf36ff09f5e5/) へ。別のエラーで困ったら [エラーメッセージ検索ツール](https://1280itsuya.github.io/devtools/error/)（貼るだけで原因と直し方を表示）も使えます。


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/aikit260628)- 原因を根本から理解するなら体系的な技術書（Amazon: [Terraform 実践](https://www.amazon.co.jp/s?k=Terraform%20%E5%AE%9F%E8%B7%B5&tag=1280itsuya22-22)）
