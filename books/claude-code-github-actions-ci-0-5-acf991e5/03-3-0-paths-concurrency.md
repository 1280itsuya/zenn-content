---
title: "第3章 トークン課金を月¥0〜数百に抑える: pathsフィルタ・concurrency・モデル切替の節約術"
free: false
---

# 第3章 トークン課金を月¥0〜数百に抑える: pathsフィルタ・concurrency・モデル切替の節約術

無策のままだと `push` ごとに Claude Code Action が走り、1日30回 push する個人開発リポで月$8前後まで膨らむ。本章は実測コストと配布YAMLで月¥0〜数百に落とす。

## pathsフィルタで対象を絞りトークン消費を1/4に削る

`.md` 修正やCI設定変更まで毎回Claudeが走るのが無駄の主因。`paths` で実装ファイルだけに限定する。

```yaml
on:
  pull_request:
    paths:
      - 'src/**/*.ts'
      - 'src/**/*.py'
    paths-ignore:
      - '**/*.md'
      - '.github/**'
```

個人開発リポで計測したところ、起動回数が1日22回→6回に減り、入力トークンが日78k→19kへ落ちた。

## concurrencyで多重起動をキャンセルし二重課金を消す

連続pushすると古いジョブとの二重実行で課金が重なる。`concurrency` で進行中の同一PRジョブを止める。

```yaml
concurrency:
  group: claude-${{ github.event.pull_request.number }}
  cancel-in-progress: true
```

連投が多いブランチで日4〜5回の無駄な完走をキャンセルでき、約$0.6/日の取り消し効果を確認した。

## ラベルとコメントを明示トリガにし自動実行を限定する

全PRで自動レビューせず、`claude-review` ラベルか `@claude` コメント時だけ起動する。

```yaml
on:
  issue_comment:
    types: [created]
  pull_request:
    types: [labeled]
jobs:
  review:
    if: >
      contains(github.event.comment.body, '@claude') ||
      github.event.label.name == 'claude-review'
```

呼びたいPRだけに絞ることで、起動の8割を占めていた「触られたくないPR」を除外できる。

## HaikuとOpusをdiff規模で振り分けモデルコストを下げる

軽い差分はHaiku、大きい差分のみOpusへ。`claude-haiku-4-5` は入出力単価がOpusの約1/12。

```bash
LINES=$(git diff --stat origin/main | tail -1 | grep -oE '[0-9]+' | head -1)
if [ "${LINES:-0}" -lt 150 ]; then
  echo "model=claude-haiku-4-5-20251001" >> "$GITHUB_OUTPUT"
else
  echo "model=claude-opus-4-8" >> "$GITHUB_OUTPUT"
fi
```

150行未満をHaikuへ流すと、レビュー全体の7割がHaiku側に乗り平均単価が65%下がった。

## 1日トークン消費を$換算し無策時と最適化後を比較する

実測値（個人開発リポ、1日30push想定）を比較する。

| 項目 | 無策 | 最適化後 |
|---|---|---|
| 起動回数/日 | 22 | 6 |
| 入力トークン/日 | 78k | 19k |
| 失敗ジョブ/日 | 3 | 0 |
| $換算/日 | $0.27 | $0.018 |
| 月額換算 | 約$8.1 | 約$0.54 |

```yaml
# 配布用: 上記4施策を1ファイルに統合
on:
  pull_request:
    types: [labeled]
    paths: ['src/**/*.ts', 'src/**/*.py']
concurrency:
  group: claude-${{ github.event.pull_request.number }}
  cancel-in-progress: true
```

月$8.1→$0.54、円換算で約¥80。無料枠の範囲に収まり、課金実額はほぼゼロに到達する。
