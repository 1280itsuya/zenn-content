---
title: "Error: Reference to undeclared resource"
emoji: "🐛"
type: "tech"
topics: ["Terraform", "インフラ", "IaC"]
published: true
---

## 発生条件

- Terraformの設定ファイル内で、存在しないリソース名や誤ったリソースタイプを参照したとき
- `resource "aws_instance" "web"` のように定義したリソースを、別の箇所で `aws_instance.app` など異なる名前で参照したとき
- モジュール分割やファイル分割後に、参照元のリソース名を変更し忘れたとき

## 原因

`Error: Reference to undeclared resource` は、Terraformが依存解決を行う際に、参照先のリソースが現在の設定内に存在しないと判断した場合に発生する。

以下のように、定義名と参照名が一致していないケースが典型的。

```hcl
# リソース定義
resource "aws_security_group" "web_sg" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id
}

# EC2インスタンス — 誤った名前で参照している
resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"

  # "app_sg" は定義されていない → Error: Reference to undeclared resource
  vpc_security_group_ids = [aws_security_group.app_sg.id]
}
```

上記では `aws_security_group.app_sg` を参照しているが、実際に定義されているのは `aws_security_group.web_sg` であるため、`terraform validate` 実行時にエラーが発生する。

タイプ名のミスも同じエラーを引き起こす。

```hcl
# 誤ったリソースタイプ
resource "aws_s3_bucket" "assets" {
  bucket = "my-assets-bucket"
}

resource "aws_cloudfront_distribution" "cdn" {
  # "aws_s3bucket" というタイプは存在しない
  origin {
    domain_name = aws_s3bucket.assets.bucket_domain_name
    origin_id   = "S3Origin"
  }
}
```

## 修正方法

リソース参照の形式は `<リソースタイプ>.<ローカル名>.<属性>` であることを確認し、定義と参照を正確に一致させる。

```hcl
# リソース定義
resource "aws_security_group" "web_sg" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id
}

# 修正後 — 定義名 "web_sg" と一致させる
resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"

  vpc_security_group_ids = [aws_security_group.web_sg.id]
}
```

```hcl
# タイプ名も修正
resource "aws_s3_bucket" "assets" {
  bucket = "my-assets-bucket"
}

resource "aws_cloudfront_distribution" "cdn" {
  origin {
    # "aws_s3_bucket" に修正
    domain_name = aws_s3_bucket.assets.bucket_domain_name
    origin_id   = "S3Origin"
  }
}
```

修正後は必ず `terraform validate` を実行して構文上の問題がないことを確認する。

```bash
terraform validate
# Success! The configuration is valid.
```

プロジェクト規模が大きくなりリソース定義が複数ファイルに分散すると、リネーム時に参照を更新し忘れるケースが増える。VSCodeの[HashiCorp Terraform拡張](https://marketplace.visualstudio.com/items?itemName=HashiCorp.terraform)を利用すると、参照先が未定義の場合にエディタ上で即座にエラーが表示されるため、`terraform validate` を実行する前に気づきやすくなる。また、`terraform plan` は `validate` より詳細なチェックを行うが、このエラーは構文レベルで検出されるため `validate` だけで十分に捕捉できる。

---

📌 このエラーの要点だけ知りたい方は [エラー即答ページ](https://1280itsuya.github.io/devtools/error/e/bad7e34a446c/) へ。別のエラーで困ったら [エラーメッセージ検索ツール](https://1280itsuya.github.io/devtools/error/)（貼るだけで原因と直し方を表示）も使えます。


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/aikit260628)- 原因を根本から理解するなら体系的な技術書（Amazon: [Terraform 実践](https://www.amazon.co.jp/s?k=Terraform%20%E5%AE%9F%E8%B7%B5&tag=1280itsuya22-22)）
