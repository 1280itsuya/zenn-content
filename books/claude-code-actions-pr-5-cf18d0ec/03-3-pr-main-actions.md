---
title: "第3章 失敗テストの自動修正PRとmainマージ前ゲートをActionsで組む"
free: false
---

以下が第3章の本文です。

## workflow_run で pytest 失敗を検知し Claude Code に渡す

CIが落ちた瞬間だけ起動するのが赤字回避の起点になる。`push`で毎回Claudeを回すと無駄実行が増えるため、テスト用ワークフローの完了イベント（`workflow_run`）に限定し、失敗時のみ起動に絞る。

```yaml
name: auto-fix
on:
  workflow_run:
    workflows: ["ci"]          # 第2章で作ったテストjob名
    types: [completed]
jobs:
  fix:
    if: >
      github.event.workflow_run.conclusion == 'failure' &&
      github.event.workflow_run.event == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.workflow_run.head_branch }}
```

`conclusion == 'failure'`を外すと成功ビルドでも起動し、後述の$11溶かし事故の半分はこの1行欠落が原因だった。

## pytest ログを 200 行に切って Claude へ食わせる

失敗ログ全文を渡すと入力トークンが膨らみ1回$0.3超えになる。`tail -n 200`で直近の失敗箇所だけ切り出し、1回の入力を平均8kトークン（約$0.024）に抑える。

```bash
- name: collect failing log
  run: |
    pytest -q 2>&1 | tail -n 200 > /tmp/fail.log || true
    echo "FAIL_LOG<<EOF" >> $GITHUB_ENV
    cat /tmp/fail.log >> $GITHUB_ENV
    echo "EOF" >> $GITHUB_ENV
```

ログを丸ごと渡した初週はClaude Sonnetで1修正$0.4前後、200行制限後は$0.05前後まで落ちた。

## anthropics/claude-code-action に max-turns を必ず付ける

暴走の正体は無制限ループだった。`max_turns`を付けず放置した結果、同一PRに修正diffが7連鎖し、1PRだけで$11課金されてジョブを手動停止した。`max_turns: 6`で1起動あたりの上限を$0.3前後に固定する。

```yaml
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          max_turns: 6
          model: claude-sonnet-4-6
          prompt: |
            次のpytest失敗ログを読み、原因を最小diffで修正せよ。
            新規依存追加・テスト削除は禁止。
            ${{ env.FAIL_LOG }}
```

`max_turns`は再発防止の最重要パラメータで、6を超えると修正の質より連鎖事故のリスクが上回った。

## 修正 diff を fix/ ブランチに出して PR を起こす

mainへ直pushさせず、必ず`fix/`プレフィックスの別ブランチへ逃がす。これでmainマージ前ゲートを人間レビューに残しつつ、Claudeの作業だけ自動化できる。

```bash
      - name: open fix PR
        run: |
          BR="fix/auto-${{ github.event.workflow_run.head_sha }}"
          git checkout -b "$BR"
          git config user.name "claude-bot"
          git commit -am "fix: auto-repair failing pytest"
          git push origin "$BR"
          gh pr create --base main --head "$BR" \
            --title "auto-fix: ${{ github.event.workflow_run.head_branch }}" \
            --body "pytest失敗をClaudeが自動修正。人間レビュー必須" \
            --label auto-fix
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## 同一ブランチ再実行ガードで $11 連鎖を止める

再発防止の決定打を入れる。`fix/`ブランチのCI失敗にまた反応すると無限ループになるため、head_branchが`fix/`で始まる場合は即終了させ、ラベル`skip-autofix`でも手動停止できる退路を用意する。

```yaml
    if: >
      github.event.workflow_run.conclusion == 'failure' &&
      !startsWith(github.event.workflow_run.head_branch, 'fix/') &&
      !contains(github.event.workflow_run.pull_requests[0].labels.*.name, 'skip-autofix')
```

この3条件（failure限定・`fix/`除外・ラベル退路）を入れてから1か月、自動修復ジョブの月額は$4.8で安定し、$11の連鎖事故は再発していない。
