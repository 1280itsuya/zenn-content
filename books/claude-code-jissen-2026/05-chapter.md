---
title: "claude -p ヘッドレス実行——スクリプトからAIを呼び出す"
free: false
---

本章では、人間が画面の前にいなくてもAIを動かす方法を扱います。鍵は `claude -p`、ヘッドレス実行です。私はこの仕組みで、記事生成からQiita/Zenn/Dev.toへの投稿までを毎日自動化しています。

### 対話モードとヘッドレスモードの違い

`claude` とだけ打つと、人間とAIが交互にやり取りする対話モードが起動します。一方 `claude -p "プロンプト"` は一度きりの実行です。

- 結果を標準出力へ書いて終了する
- 途中で人間に質問しない
- 終了コードで成否を判定できる

`grep` や `curl` と同じ「普通のコマンド」としてAIを扱えるわけです。なおヘッドレス実行中は許可の確認ができないため、ツールは `--allowedTools` か settings.json(第3章)で事前に許可します。

### 標準入出力の基本形

基本形は2つです。

```powershell
# 引数でプロンプトを渡す
claude -p "配列の重複除去をPowerShellで1行で"
# 標準入力でデータを渡す
Get-Content error.log | claude -p "原因を3行で要約して"
```

「データはパイプ、指示は引数」と分担します。両方はまとめて入力になります。

### 最小パイプライン——生成→保存→次工程

私の記事生成パイプラインの骨格を、写経できる形まで削ったものです。

```powershell
$ErrorActionPreference = "Stop"
$theme  = "PowerShellのログローテーション"
$outDir = "C:\pipeline\out"
New-Item -ItemType Directory -Force $outDir | Out-Null

$prompt = @"
テーマ「$theme」で技術記事をMarkdownで書いてください。
見出しは##、コード例1つ以上。本文だけを出力。
"@

$article = $prompt | claude -p
if ($LASTEXITCODE -ne 0) { throw "claude -p 失敗" }

$path = Join-Path $outDir "article.md"
$article | Out-File $path -Encoding utf8
# python .\post_article.py $path   # 次工程へ渡す
```

ポイントは3つです。`$LASTEXITCODE` で成否を必ず確認すること。`-Encoding utf8` を忘れないこと(PowerShell 5.1の既定はUTF-16です)。「本文だけを出力」と明示することです。これを守らせないと前置きの挨拶が混ざり、後工程が壊れます。

### 出力形式の制御とパース

機械的に扱うなら `--output-format json` が安全です。

```powershell
$obj = claude -p "TSの利点を3つ、JSON配列だけで" --output-format json | ConvertFrom-Json
if ($obj.is_error) { throw "生成エラー" }
$obj.result   # 本文
```

本文は `result`、エラー有無は `is_error` に入り、本文とメタ情報が分離されます。本文自体をJSONにしたい時は「JSONのみ。コードフェンス禁止」と指定します。それでも囲んで返すことがあるので、フェンスを剥がす前処理を必ず入れています。AIの出力は「たまに裏切る」前提で受けます。

### リトライとタイムアウトの設計

生成は時々失敗します。私は次のパターンで包んでいます。

```powershell
function Invoke-ClaudeSafe {
  param([string]$Prompt, [int]$Retry = 3, [int]$TimeoutSec = 300)
  for ($i = 1; $i -le $Retry; $i++) {
    $job = Start-Job { param($p) $p | claude -p --output-format json } -ArgumentList $Prompt
    if (Wait-Job $job -Timeout $TimeoutSec) {
      $obj = Receive-Job $job | ConvertFrom-Json
      Remove-Job $job -Force
      if (-not $obj.is_error) { return $obj.result }
    } else { Stop-Job $job; Remove-Job $job -Force }
    Start-Sleep -Seconds (15 * $i)
  }
  throw "リトライ上限"
}
```

重要なのは「タイムアウトなしで claude -p を呼ばない」ことです。

### 落とし穴1: パイプ待ちで数時間ハング

私のパイプラインは、ある朝から沈黙しました。`claude -p` へのパイプ渡しが応答待ちのまま止まり、CPUほぼゼロで6時間以上ハングしていました。後続の処理もすべてその後ろで詰まっていました。対策は3つです。

1. 全呼び出しにタイムアウトを付ける(前掲のジョブ方式)
2. 開始時刻をロックファイルへ記録し、異常に長ければ強制終了して再実行
3. 成果物ファイルの有無で外から健全性を確認できる設計にする

ハングはエラーを出しません。検知の仕組みを最初から入れる必要があります。

### 落とし穴2: 非対話環境での認証プロンプト待ち

タスクスケジューラ(第8章)からの実行は、画面のない環境で走ります。そこで認証ダイアログが出ると、誰も押せないまま待ち続けます。私は `git push` がGitHubの認証ダイアログを開き、約2時間止まった経験があります。回避策は「対話が発生しない状態を事前に作る」ことです。

```powershell
$env:GIT_TERMINAL_PROMPT = "0"  # gitの対話プロンプトを禁止(即エラー化)
```

- Claude Codeのログインは先に対話モードで済ませておく
- gitはトークンを資格情報に保存し、ダイアログ不要にする
- スケジューラに載せる前に手動で1回通す

「止まって待つ」より「即エラーで落ちる」ほうが自動化では健全です。エラーは拾えますが、待機は何も残しません。

### まとめ

`claude -p` はAIを普通のコマンドに変えます。JSON出力で本文とメタ情報を分離し、タイムアウトとリトライで包み、認証は事前に済ませる。これがヘッドレス運用の土台です。次章では、ここに品質チェックを差し込むhooksを扱います。
