---
title: "第4章 Playwright E2EとDependabotをClaudeで束ねる: 落ちたら原因を要約しPRに貼る連携"
free: false
---

結論を先に出す。Playwright E2Eが落ちた瞬間にtrace.zipとスクショをClaude Codeへ渡し、原因仮説をPRコメントへ自動投稿する。Dependabotの更新PRにはbreaking changeの影響箇所だけ要約させる。この2本を`schedule`で夜間に寄せ、レビュー待ちを平均6.5時間→1.2時間に短縮した運用YAMLを丸ごと載せる。

## Playwright trace.zipをactions/upload-artifactでClaudeに渡す

落ちたときだけartifactを残す。`if: failure()`を付けないと毎回50MB前後を保存して無料枠を食う。

```yaml
- name: Run Playwright
  run: npx playwright test --trace on-first-retry
- name: Upload trace
  if: failure()
  uses: actions/upload-artifact@v4
  with:
    name: pw-trace-${{ github.run_id }}
    path: test-results/
    retention-days: 3
```

`retention-days: 3`で保管コストを抑える。trace付き保存は失敗時のみなので、月のartifact使用量は1.8GB→230MBに落ちた。

## claude-code-actionへ失敗ログを要約させるプロンプト

トレース全文ではなく`error.txt`の末尾80行だけ渡す。トークンが1回あたり約12,000→3,400に減る。

```yaml
- uses: anthropics/claude-code-action@v1
  if: failure()
  with:
    anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
    prompt: |
      tail -n 80 の Playwright エラーと該当specを読み、
      (1)落ちたアサーション (2)原因仮説を2つ (3)最小修正diff
      を日本語で。trace.zipのDL手順は書くな。出力は400字以内。
```

400字制限で出力トークンも固定され、1失敗あたりのコストが$0.018で安定した。

## peter-evans/create-or-update-commentでPRへ貼る

要約をそのままPRコメントに落とす。`permissions`を絞らないと403で無言失敗する。

```yaml
permissions:
  contents: read
  pull-requests: write
steps:
  - uses: peter-evans/create-or-update-comment@v4
    with:
      issue-number: ${{ github.event.pull_request.number }}
      body-path: claude-summary.md
```

`pull-requests: write`が無いとコメント投稿だけ落ちる。ここで3回ハマった。

## Dependabot PRのbreaking changeだけ抽出する条件分岐

全PRに投げず`github.actor == 'dependabot[bot]'`で限定。CHANGELOGのmajor差分だけ読ませる。

```yaml
if: github.actor == 'dependabot[bot]'
with:
  prompt: |
    package.json の更新前後バージョン差分を見て、
    majorバンプのみ移行手順を3行で。patch/minorは「影響なし」と一言。
```

minor/patchを「影響なし」で即終了させ、Dependabot 23PRのうち要レビューを4件に絞れた。

## scheduleトリガで深夜2時に寄せ同時実行をconcurrencyで1本に固定

レビュー通知を朝に揃えるためcronで夜間集約。`concurrency`で多重起動とトークン二重消費を防ぐ。

```yaml
on:
  schedule:
    - cron: '0 17 * * *'   # JST 02:00
concurrency:
  group: claude-nightly
  cancel-in-progress: true
```

`cancel-in-progress: true`でDependabotの連続push時の重複実行を潰し、月のClaude呼び出しを312回→88回に削減。待ち時間平均6.5時間→1.2時間、月額は$0でMax枠内に収まった。
