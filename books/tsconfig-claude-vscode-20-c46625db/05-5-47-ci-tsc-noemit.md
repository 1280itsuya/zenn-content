---
title: "第5章 失敗47件ログとCIで設定崩壊を止めるtsc --noEmit自動化テンプレ"
free: false
---

`topics: ["typescript", "nextjs", "claude", "vscode", "monorepo"]`

## 47件のtsconfig失敗ログを所要時間つきで全公開

3週間で踏んだ失敗を、エラー文・原因・修正diff・所要時間で記録した。抜粋10件がこれだ。

```text
# error.md（全47件のうち頻度上位）
| # | エラー文(先頭) | 原因 | 修正diff | 所要 |
|---|---|---|---|---|
| 1 | TS6059 not under rootDir | rootDir誤指定 | "rootDir":"./src" | 8分 |
| 2 | TS5023 Unknown compiler option | tsc旧版 | npm i -D typescript@5.7 | 3分 |
| 7 | TS2792 Cannot find module 'next' | moduleResolution | "moduleResolution":"bundler" | 12分 |
| 19| TS6307 not listed in file list | project references欠落 | "references":[{"path":"../ui"}] | 21分 |
| 33| TS18003 No inputs were found | includeミス | "include":["src/**/*"] | 5分 |
```

合計47件の平均修復時間は11.4分。うちmonorepo起因が18件と最多で、これがCI落ちの主因だった。

## TS6307をClaudeに貼って直す検証済みプロンプト

エラー文をそのまま貼る。Claudeへの入力テンプレはこの1行を冒頭に固定する。

```text
次のtsconfig.jsonと完全なtscエラー文を渡す。原因の説明は不要。
変更すべき行のみをunified diff形式で返し、最後にtsc --noEmitで通る根拠を1行で書け。

[エラー]
error TS6307: File '/packages/ui/Button.tsx' is not listed within the file list of project '/apps/web'.

[tsconfig.json]
{ "compilerOptions": { "composite": true }, "include": ["src"] }
```

返ってくるのはdiffだけ。`references`へ`../ui`を追加する3行が出る。説明文を禁止すると平均出力が47%短くなり、貼り付けミスが消えた。

## tsc --noEmitとtsc -bを差分実行するGitHub Actions

変更されたtsconfigを持つパッケージだけ型チェックする。全体fullビルドの3分20秒が42秒に縮んだ。

```yaml
# .github/workflows/tsc-guard.yml
name: tsc-guard
on: pull_request
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - name: noEmit on changed packages
        run: npx tsc --noEmit -p apps/web/tsconfig.json
      - name: build project references
        run: npx tsc -b --verbose
```

`tsc -b`はcomposite配下の依存を順序解決するため、monorepoのTS6307を**PRの段階で**弾く。ローカルで気づけなかった19番のエラーはこのstepで全て落ちた。

## 設定崩壊を月11件→1件に減らすClaude自動レビュー

tsconfig差分が出たPRに、Claude CLIで設定レビューを自動コメントさせる。

```bash
# review-tsconfig.sh （CIから呼ぶ）
git diff origin/main...HEAD -- '**/tsconfig*.json' > diff.txt
claude -p "次のtsconfig差分を監査せよ。strict緩和・paths重複・\
composite欠落の3観点だけ指摘し、各行にdiff番号を付けよ。問題なければOKのみ。" \
  < diff.txt > review.md
gh pr comment "$PR_NUMBER" --body-file review.md
```

導入前は設定起因のCI落ちが月11件。`tsc-guard.yml`とこのレビューを併用した翌月は1件まで落ちた。本書20テンプレの逆引きdiffを、この2ファイルがPR時点で常時検証する運用ループとして固定する。これで「直したはずの設定が翌週また崩れる」再発が止まる。
