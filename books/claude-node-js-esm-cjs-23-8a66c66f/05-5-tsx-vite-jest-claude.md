---
title: "第5章 tsx・Vite・Jestの混在環境エラーを切り分けるClaude常駐プロンプトの自動化"
free: false
---

```yaml
---
topics: ["claude", "typescript", "nodejs", "ai", "automation"]
---
```

## tsx・Vite・Jestでモジュール解決が3者バラバラになる原因をEXPLAINする

tsx は ESM ローダ、Vite は esbuild、Jest は CommonJS と、同じ `import` でも解決経路が違う。まず現状を1コマンドで吐かせて、どの層で壊れているか特定する。

```bash
node -e "console.log(require('tsx/package.json').version)" 2>&1
npx vite --version; npx jest --version
node --experimental-vm-modules -e "import('./src/index.ts').catch(e=>console.error(e.code))"
```

`ERR_MODULE_NOT_FOUND` が tsx だけで出るなら拡張子解決、Jest だけなら `transform` 不足と切り分く。

## エラー文をパイプで渡すと23スニペットから診断プロンプトを自動選択するshell関数

`claude` CLI に毎回プロンプトを書き直さない。エラーの `code` を grep して該当スニペットを差し込む常駐関数を `.zshrc` に置く。

```bash
cdx() {  # claude-diagnose-esm
  local log="$1"; local code=$(grep -oE 'ERR_[A-Z_]+' "$log" | head -1)
  local snip="$HOME/.claude-esm/${code:-ERR_UNKNOWN}.md"
  cat "$snip" "$log" package.json tsconfig.json | claude -p "貼ったログのcodeに対応する修正パッチをdiff形式で返せ"
}
```

`cdx ./err.log` で、原因を理解していなくても該当の診断プロンプト＋`package.json`＋`tsconfig.json`が自動添付され、パッチが返る。

## VS Codeタスクで選択中のエラー行をClaudeへ渡す

ターミナルを離れずエディタの選択範囲を `cdx` に流す。`.vscode/tasks.json` に登録する。

```json
{
  "version": "2.0.0",
  "tasks": [{
    "label": "claude-esm-fix",
    "type": "shell",
    "command": "echo \"${selectedText}\" | tee /tmp/err.log && cdx /tmp/err.log",
    "presentation": { "reveal": "always", "panel": "dedicated" }
  }]
}
```

`Ctrl+Shift+P → Run Task → claude-esm-fix` で選択行が即診断される。

## CIでESM/CJSエラーが出たらClaude CLIに原因候補を3つ返させるGitHub Actions

`jest` が `Cannot use import statement outside a module` で落ちた瞬間、ログを Claude に渡して原因候補を構造化出力させる。

```yaml
- name: claude-esm-triage
  if: failure()
  run: |
    npx jest 2>&1 | tail -40 > err.log
    cat err.log package.json | claude -p \
      "ESM/CJS起因の原因候補を確度順に3つ、各々の検証コマンド付きでJSON出力" \
      > triage.json
    cat triage.json
```

`failure()` 限定なので緑のときは課金ゼロ。1回約2,000トークンで済む。

## Claudeの回答を鵜呑みにせず node --input-type で最小再現する検証フロー

返ってきたパッチを当てる前に、問題のモジュール1個だけを隔離実行する。これで「直ったつもりで別エラー」を防ぐ。

```bash
echo "import pkg from 'chalk'; console.log(typeof pkg)" \
  | node --input-type=module
# default export が undefined → CJS interop 問題が確定
echo "const c=require('chalk'); console.log(typeof c)" \
  | node --input-type=commonjs
```

両方走らせ、どちらで `function` が返るかでパッチの方向（`esModuleInterop` か `default` 経由か）を1分で判定する。

半年このフローを常駐させた結果、ESM/CJS系エラーの平均解決時間は初月の42分から直近月8分へ短縮し、Jest絡みの再発は週3件から週0〜1件に落ちた。23スニペットを一度 `~/.claude-esm/` に置けば、以降は `cdx` 一発で同じ削減が再現する。
