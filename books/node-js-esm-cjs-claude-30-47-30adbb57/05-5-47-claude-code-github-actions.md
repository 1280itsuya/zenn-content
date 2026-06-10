---
title: "第5章: 47プロンプトをClaude Code+GitHub Actionsで自動診断パイプライン化する"
free: false
---

## `.env`にANTHROPIC_API_KEYを置きClaude Codeを非対話起動する

CIで動かす前提なら、APIキーはリポジトリではなくGitHub Secretsに置く。ローカル検証用の`.env`はこう書く。

```bash
# .env (ローカル検証専用 / .gitignore必須)
ANTHROPIC_API_KEY=sk-ant-xxxx
CLAUDE_MODEL=claude-opus-4-8
```

非対話でエラーログを食わせる起動はこの一行。第3章の47プロンプトを`prompts/esm-cjs.md`に切り出しておき、`-p`に注入する。

```bash
cat build.log | claude -p "$(cat prompts/esm-cjs.md) 上記辞書で次のエラーを診断しdiffで修正案を出せ" \
  --model "$CLAUDE_MODEL" --output-format json > diag.json
```

## workflow.ymlでビルド失敗時のみ診断を起動する

`if: failure()`を付け、ビルドが緑のときはClaudeを呼ばない。これでAPIコストを失敗回数ぶんに限定する。

```yaml
# .github/workflows/diagnose.yml
name: ai-diagnose
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build 2>build.log
      - name: Claude diagnose
        if: failure()
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          npm i -g @anthropic-ai/claude-code
          cat build.log | claude -p "$(cat prompts/esm-cjs.md) このログを症状別辞書で診断しdiff出力" \
            --model claude-opus-4-8 --output-format json > diag.json
```

`ERR_REQUIRE_ESM`が出た実ログを食わせると、Claudeは「`package.json`に`"type":"module"`、または`require()`を動的`import()`へ」と返し、次のdiffまで生成する。

```diff
- const chalk = require("chalk");
+ const chalk = (await import("chalk")).default;
```

## diag.jsonをパースしてPR本文を組み立てる

`--output-format json`の`result`フィールドだけ抜き、PR本文へ渡す。生ログ全文は載せず診断と差分のみにすると、レビュー時間が短い。

```bash
jq -r '.result' diag.json > pr_body.md
gh pr create --title "fix: ESM/CJSエラー自動診断" --body-file pr_body.md --base main --head ai/diag-${{ github.run_id }}
```

## 承認ゲート: labelが付くまでmergeを禁止する

誤PR対策の核心はここ。Claudeが起票したPRは`needs-human`ラベル付きで生成し、人がレビューし`approved`へ貼り替えるまで`auto-merge`を発火させない。

```yaml
  gate:
    if: contains(github.event.pull_request.labels.*.name, 'approved')
    runs-on: ubuntu-latest
    steps:
      - run: gh pr merge ${{ github.event.number }} --squash --auto
        env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
```

これで「AIが勝手にmergeした」事故は構造的に起きない。承認はラベル1クリックで済む。

## Slack通知と前月比の実測値で運用を回す

診断PRが立ったらSlackへ飛ばす。`diag.json`の要約3行だけ送ると、スマホで可否判断できる。

```bash
curl -s -X POST "$SLACK_WEBHOOK" -H 'Content-type: application/json' \
  -d "{\"text\":\"🔧 ESM診断PR: $(jq -r '.result' diag.json | head -3)\"}"
```

筆者環境のモノレポ7パッケージで計測したところ、CI失敗1件あたりの調査+修正時間は前月平均42分→このパイプライン導入後は承認込み9分へ。月18件で換算すると約9.9時間の削減になった。コストはClaude呼び出しが失敗時のみで月$3.1に収まっている。手で辞書を引く運用から、落ちたら診断PRが待っている運用へ移ると、固定費削減の感覚で時間が戻ってくる。
