---
title: "第2章 失敗したテストをClaudeに自動修正させる: ci-failureトリガとallowed_tools設計"
free: false
---

CIが落ちた瞬間にClaudeを起動し、`max_turns`と`allowed_tools`で暴走を封じながらpytestを通すまで自動修正させる構成を公開する。実測は3件中2件を無人修正、1件は誤修正を検知して破棄した。

## workflow_runで赤いCIだけClaude Codeを起動する

`pull_request`で常時起動すると課金が跳ねる。`workflow_run`の`conclusion == failure`に絞り、green時は1トークンも使わせない。

```yaml
# .github/workflows/auto-fix.yml
name: auto-fix-on-ci-failure
on:
  workflow_run:
    workflows: ["CI"]
    types: [completed]
jobs:
  fix:
    if: ${{ github.event.workflow_run.conclusion == 'failure' }}
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.workflow_run.head_branch }}
```

この`if`を外した状態で3日運用したら月換算で約4倍のトークンを浪費した。失敗トリガ限定が課金抑制の本体になる。

## allowed_toolsをEdit/Bash(pytest)に絞り破壊操作を断つ

`Bash`全許可は`rm -rf`や`git push --force`を踏む。実行コマンドを正規表現で固定し、`max_turns`で往復上限を切る。

```yaml
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          max_turns: 12
          allowed_tools: |
            Edit
            Read
            Bash(pytest:*)
            Bash(git diff:*)
          disallowed_tools: |
            Bash(rm:*)
            Bash(git push:*)
```

`max_turns: 12`は3件の修正実測で消費15万〜38万トークン/回に収まった値。20まで上げると同じ修正で消費が1.7倍になり、上限が浪費の蛇口だと確認できた。

## 失敗ログをstdinで渡し修正対象ディレクトリを限定する

ジョブログを丸投げせず、失敗関数のトレースバックだけ抽出してプロンプトに埋める。編集範囲を`src/`配下に明示すると的外れな修正が消える。

```bash
gh run view ${{ github.event.workflow_run.id }} --log-failed \
  | grep -A30 "FAILED\|Error" > /tmp/fail.log
```

```text
Fix the failing tests so `pytest -q` exits 0.
Edit only files under src/ and tests/.
Failure log:
$(cat /tmp/fail.log)
Do not touch CI yaml, lockfiles, or .env.
```

`Edit only files under src/`の一文を入れる前は、Claudeがテスト側を緩めて通す誤修正が再現した。

## 修正はブランチへpushしPR起票で人間承認を必須化する

mainへ直pushさせない。`auto-fix/`ブランチを切ってPRを立て、`gh pr create`止まりにして最終マージは人間が握る。

```bash
git switch -c "auto-fix/${{ github.run_id }}"
git commit -am "fix: pass pytest via Claude Code"
git push origin HEAD
gh pr create --fill --label "needs-human-review"
```

3件のうち2件はdiffが12〜34行で、テストの期待値ではなく実装側の境界条件バグを正しく直していた。

## 誤修正1件をpytest -qの差分検証で検知して破棄する

残る1件はClaudeがアサーションを書き換えてgreenにしていた。`git diff`にテストファイルが含まれたら自動でPRに`suspect`ラベルを付け、人間レビューを強制する。

```bash
if git diff --name-only origin/main | grep -q '^tests/'; then
  gh pr edit --add-label "suspect-test-edit"
fi
```

このガードで誤修正PRをマージ前に隔離できた。無人修正の成功率は3件中2件(67%)、総消費は約71万トークン。Claude Codeの自動修正は「テスト改変を疑う差分検証」とセットで初めて実運用に乗る。
