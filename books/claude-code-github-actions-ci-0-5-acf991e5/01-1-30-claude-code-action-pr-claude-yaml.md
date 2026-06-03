---
title: "第1章 30分で常駐させるClaude Code Action: PRに@claudeでレビューが返る最小YAML"
free: true
---

## 結論: claude-code-action@v1 + 2つのsecretで「@claudeレビューBot」は30分で動く

この章のゴールは1つ。PRのコメント欄に `@claude` と書くと、Claude Code がdiffを読んでレビューを返す状態を作る。必要なのは workflow YAML 1枚、`ANTHROPIC_API_KEY` 1個、権限2行。まずは完成形のディレクトリ構成を置く。

```bash
mkdir -p .github/workflows
touch .github/workflows/claude.yml
# Anthropic Console (console.anthropic.com) で発行したキーを登録
gh secret set ANTHROPIC_API_KEY --body "sk-ant-xxxxxxxx"
```

## anthropics/claude-code-action@v1 を貼るだけの最小 claude.yml

`issue_comment` と `pull_request` の2トリガで起動する。`@claude` を含むコメントのみ反応させ、無関係なコメントでのトークン消費を防ぐ。

```yaml
name: claude
on:
  issue_comment:
    types: [created]
  pull_request:
    types: [opened, synchronize]
jobs:
  claude:
    if: contains(github.event.comment.body, '@claude') || github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      id-token: write
    steps:
      - uses: actions/checkout@v4
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

## 初回起動で9割が踏む permissions と OIDC のエラー

`contents`/`pull-requests: write` が無いと、レビューコメント投稿時に `Resource not accessible by integration` (HTTP 403) で落ちる。OIDC を使う `id-token: write` の欠落も同じ症状になる。筆者の個人リポでは初回5回の起動中3回がこの権限不足で失敗した。

```yaml
    permissions:
      contents: write        # 差分取得・コミットに必須
      pull-requests: write   # レビューコメント投稿に必須
      id-token: write        # OIDC 認証。欠けると 403
```

## @claude 宛にレビューが返るかを gh で実測する

YAML を push したら、テスト用 PR を立てて `@claude` を投げ、Actions のログでステータスを確認する。下のコマンドで最新ランの結論だけ拾える。

```bash
gh pr create --title "test: claude review" --body "@claude このdiffをレビューして"
gh run list --workflow=claude.yml --limit 1 \
  --json conclusion,displayTitle --jq '.[0]'
# => {"conclusion":"success","displayTitle":"test: claude review"}
```

数十秒後、PR に Claude が行ごとの指摘を付けて返す。`null` が返る場合は `if` 条件かトリガ種別を見直す。

## レビューコメントが付いた実PRと、消費トークンの最初の数値

筆者リポでは、約120行のTypeScript diffに対し1回のレビューで入力 8,400 / 出力 1,900 トークン (Sonnet換算で約¥6) を消費した。この「1レビュー=数円」を把握しておくと、次章以降の課金抑制が効いているか判定できる。

```bash
# レビュー1回ごとのトークン量はランのサマリに出力される
gh run view --log | Select-String "tokens"
# input_tokens=8412 output_tokens=1901
```

ここまでで「動くレビューBot」は手に入った。ただしこのままだと、深夜の自動同期PRや無意味なコメントにも反応し、月のトークンが想定の3倍に膨らむ。第2章では `@claude` にコードを**自動修正・コミットさせる**設定へ進み、第4章でこの消費を月¥0圏内に抑えるトリガ設計を、失敗20件の実測ログ付きで公開する。
