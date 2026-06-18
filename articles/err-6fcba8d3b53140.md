---
title: "go: cannot find main module"
emoji: "🐛"
type: "tech"
topics: ["Go", "Golang", "エラー解決"]
published: true
---

## 発生条件

- `go run`・`go build`・`go test` などの `go` コマンドを、`go.mod` が存在しないディレクトリで実行したとき
- `go.mod` を持つプロジェクトのルートではなく、その **外側** のディレクトリで作業しているとき
- GOPATH モード時代のコードをそのままコピーし、モジュール初期化をせずに実行しようとしたとき

## 原因

`go: cannot find main module` は、Go がカレントディレクトリおよびその親ディレクトリを再帰的に走査しても `go.mod` を見つけられなかった場合に出力されます。Go 1.16 以降はモジュールモードがデフォルトであるため、`go.mod` の不在は即エラーになります。

以下は最小再現手順です。

```sh
# 新しい空ディレクトリを作成して移動
mkdir /tmp/demo && cd /tmp/demo

# go.mod を作らずにいきなり実行しようとする
cat > main.go << 'EOF'
package main

import "fmt"

func main() {
    fmt.Println("hello")
}
EOF

go run main.go
# => go: cannot find main module; see 'go help modules'
```

`go.mod` が無い状態では Go ツールチェーンがモジュールルートを特定できず、依存解決もビルドも行えません。エラーメッセージ末尾の `see 'go help modules'` が示すとおり、モジュールの初期化が必要です。

## 修正方法

`go mod init <module>` を実行して `go.mod` を生成します。`<module>` にはモジュールパスを指定します。GitHub でホストする場合は `github.com/<ユーザー名>/<リポジトリ名>` 形式が一般的です。個人ローカル実験なら任意の文字列で構いません。

```sh
# モジュールを初期化する（module パスは実態に合わせて変える）
go mod init github.com/yourname/demo

# go.mod が生成されたことを確認
cat go.mod
# module github.com/yourname/demo
#
# go 1.22

# 改めて実行
go run main.go
# => hello
```

`go mod init` は **プロジェクトルートで一度だけ** 実行すれば十分です。サブパッケージを追加しても `go.mod` を再作成する必要はありません。外部パッケージを `import` した場合は続けて `go mod tidy` を実行し、`go.sum` を生成してください。

既存プロジェクトをクローンしたのに `go.mod` が見当たらない場合は、リポジトリのルートではなくサブディレクトリに移動してしまっている可能性があります。`ls go.mod` でファイルの有無を確認し、見つからなければ `git rev-parse --show-toplevel` でリポジトリルートへ移動してから再実行してください。

---

📌 このエラーの要点だけ知りたい方は [エラー即答ページ](https://1280itsuya.github.io/devtools/error/e/65d69c24c005/) へ。別のエラーで困ったら [エラーメッセージ検索ツール](https://1280itsuya.github.io/devtools/error/)（貼るだけで原因と直し方を表示）も使えます。


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/aikit260618)- 原因を根本から理解するなら体系的な技術書（Amazon: [Go言語 実践 入門](https://www.amazon.co.jp/s?k=Go%E8%A8%80%E8%AA%9E%20%E5%AE%9F%E8%B7%B5%20%E5%85%A5%E9%96%80&tag=1280itsuya22-22)）
