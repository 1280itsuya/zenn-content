---
title: "第5章: Claude Codeでtsconfigエラーを一括修正しGitHub ActionsのCIで再発を止める"
free: false
---

## 結論: `tsc --noEmit` の出力をそのまま Claude Code に渡せば47件は3分で消える

この章のゴールは、第1〜4章で作った逆引き知識を「Claude Code への入力」に変換し、エラー文の貼り付けからCIブロックまでを一本の動線にすること。まず手元の全エラーを機械可読な形で吐き出す。

```bash
# 整形なしの生エラーをそのままファイルに保存（色コードを無効化）
NO_COLOR=1 npx tsc --noEmit --pretty false 2>&1 | tee tsc-errors.log
wc -l tsc-errors.log   # => 47 行前後ならこの章の対象
```

`--pretty false` を付けるのが要点で、これで `src/api.ts(12,5): error TS2307: Cannot find module './foo'` のような1行1エラー形式になり、AIが行番号を取り違えなくなる。

## Claude Code 一括修正プロンプト雛形（誤修正を防ぐ3制約）

`claude` CLI に `tsc-errors.log` を食わせる。雛形には「触ってよい範囲」を必ず明記する。

```bash
claude -p "$(cat <<'EOF'
添付の tsc-errors.log は Vite+ESM プロジェクトの型/設定エラー一覧です。
制約:
1. 変更してよいのは tsconfig.json と *.ts の import 文のみ。ロジックは変えない
2. moduleResolution は "bundler" を維持（"node" に戻さない）
3. 1ファイルごとに「TSコード→原因→修正diff」を出してから適用する
tsc-errors.log を読み、47件を上から順に修正してください。
EOF
)" tsc-errors.log
```

制約3により、Claude Code は `TS2307→拡張子 .js 補完`、`TS1259→esModuleInterop 追加` のように逆引き根拠を添えて修正するため、後段のレビューが速くなる。

## diffレビュー: `git add -p` で危険な3パターンだけ弾く

AIの修正を丸呑みしない。`paths` の上書きや `strict` の無効化など、エラーは消えるが品質が下がる修正をここで止める。

```bash
git diff tsconfig.json
# 以下が混入していたら却下（rg で一括検出）
rg -n '"strict":\s*false|"skipLibCheck":\s*true|"noImplicitAny":\s*false' tsconfig.json

# 問題なければ対話的に1ハンクずつ承認
git add -p
```

`strict: false` で47件を消すのは「修正」ではなく「隠蔽」。この3パターンを検出したらそのハンクだけ `git checkout -- tsconfig.json` で戻し、Claude Code に「strictは維持して再修正」と差し戻す。

## 人手ゲート: `vite build` が通って初めてコミットする

`tsc --noEmit` が0件でも、Vite のESMバンドルで落ちるケース（`.js` 拡張子忘れの動的import等）が残る。コミット前ゲートを1行スクリプトにする。

```bash
# gate.sh — 両方通ったときだけ exit 0
set -e
npx tsc --noEmit --pretty false
npx vite build
echo "gate passed: tsc 0 errors / vite build OK"
```

```bash
bash gate.sh && git commit -m "fix: tsconfig errors (47) via Claude Code"
```

`set -e` でどちらかが失敗すれば `&&` の右が実行されず、未検証コードがコミットされない。

## GitHub Actions で `tsc` と `vite build` を走らせPRをブロック

最後に再発防止。同じ2コマンドをCIに移植し、失敗したPRをマージ不可にする。

```yaml
# .github/workflows/typecheck.yml
name: typecheck
on:
  pull_request:
    branches: [main]
jobs:
  tsc-and-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci
      - run: npx tsc --noEmit --pretty false
      - run: npx vite build
```

このワークフローを必須化すれば、tsconfigエラーが本番に漏れる経路が物理的に閉じる。GitHub の Settings → Branches → Branch protection rules で `tsc-and-build` を **Require status checks to pass** に追加すると、緑にならない限り Merge ボタンが押せなくなる。第6章では、このCIにキャッシュを効かせて実行時間を90秒→20秒に短縮する。
