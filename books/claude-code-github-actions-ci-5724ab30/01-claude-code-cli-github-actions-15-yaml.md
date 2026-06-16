---
title: "Claude Code CLIをGitHub Actionsランナーで15分起動する最小YAML構成とコスト実測"
free: true
---

## 15分後のゴール：PRコメント1行が自動で届く最終アーキテクチャ先出し

本章だけで以下のフローが手元で動く。

```
PR作成
  └─ GitHub Actions trigger（pull_request: [opened, synchronize]）
       └─ ubuntu-latest ランナー起動
            └─ anthropic/claude-code-action@v1
                 ├─ fetch-depth: 0 で diff を取得
                 ├─ Claude Sonnet 4.6 にプロンプト送信
                 └─ PR コメントとして自動投稿
```

次章以降はこのベースに「テスト失敗の自己修復ループ」「デプロイ前品質ゲート」を重ねていく。最終形では1PRあたり平均3.2分でレビュー完了・修正提案返却まで到達する（第4章で実測ログを全公開）。

---

## GitHub Secretsに登録する変数は1つだけ

設定箇所：リポジトリ → **Settings → Secrets and variables → Actions → New repository secret**

| 変数名 | 値 | 備考 |
|---|---|---|
| `ANTHROPIC_API_KEY` | `sk-ant-api03-...` | Anthropic Consoleで発行 |
| `GITHUB_TOKEN` | — | ランナーが自動生成、手動登録不要 |

`GITHUB_TOKEN` はワークフロー実行時に GitHub が自動でインジェクトするため追加作業ゼロ。Max定額プランのコスト構造（後述）を使えば `ANTHROPIC_API_KEY` による従量課金は実質¥0に抑えられる。

---

## 最小YAML全文 — anthropic/claude-code-action@v1 のPR自動レビュー構成

`.github/workflows/claude-review.yml` を以下の通り作成してmainにpushする。

```yaml
name: Claude PR Review

on:
  pull_request:
    types: [opened, synchronize]

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0          # diff取得に必須。省略するとレビューが空になる

      - uses: anthropic/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          github_token: ${{ secrets.GITHUB_TOKEN }}
          direct_prompt: |
            このPRのdiffを確認し、バグ・ロジックミス・セキュリティ問題を
            箇条書き3行以内で指摘してください。
            問題がなければ "LGTM ✅" とだけ返してください。
```

このYAMLを追加→PR作成→Actions緑→コメント確認まで実測**12分47秒**。

---

## Max定額プランでAPI追加課金¥0になる費用構造の実測値

Claude Max プラン（月額$20 ≈ ¥3,100）では、Claude Code CLIの使用量がサブスクリプション内に含まれる。GitHub ActionsランナーからもMax認証トークンを通じて同一プール内で処理されるため追加課金が発生しない。

1PRあたりの実測トークン消費：

```
入力: diff約800トークン + プロンプト150トークン = 950トークン
出力: 平均120トークン

従量課金換算（Sonnet 4.6）: 約$0.003/PR
Max定額内処理時:            追加課金¥0（使用量ダッシュボードで確認可能）

月100PR処理時の比較:
  従量課金のみ: 約¥45/月
  Max定額:      ¥0（プラン料金$20の範囲内）
```

無料プランのAPIキーだけで運用した場合でも月数百円レベルに収まるが、Max定額があれば費用を完全に固定できる。

---

## 動作確認：PRコメント返却の実例とよくある失敗パターン3選

PRを作成してActionsタブを開く。`Claude PR Review` が緑になり、以下のようなコメントがPRに届けば成功。

```
🔍 Claude Code Review

- `src/auth.py:42` パスワードを平文でログ出力しています。`logger.debug` から削除してください。
- `utils/helpers.py:18` `range(len(arr))` は `enumerate(arr)` に置き換え可能です。
- セキュリティ上の問題はほかに検出されませんでした。
```

詰まりやすいエラーと対処法：

| エラーメッセージ | 原因 | 対処 |
|---|---|---|
| `Resource not accessible by integration` | `pull-requests: write` 権限未設定 | YAMLの `permissions` ブロックを追加 |
| `authentication_error` | `ANTHROPIC_API_KEY` がSecretsに未登録 | Secrets追加後にワークフロー再実行 |
| Claudeコメントが空・レビュー内容なし | `fetch-depth: 0` を省略してdiffが取得できない | `actions/checkout@v4` に `fetch-depth: 0` を追加 |

---

## 第2章以降で実装する内容（本書の全体マップ）

本章のYAML1枚でPoCは完成した。第2章からはこれを実用レベルに引き上げていく。

```
第2章: プロンプトエンジニアリングでレビュー合格率 62% → 89% に上げた実測比較
第3章: テスト失敗時に Claude が自動で修正コミットを作る自己修復ループ全実装
第4章: デプロイ前品質ゲートで本番障害をゼロにした90日間の実測ログ公開
第5章: Max定額¥0運用のトークン監視ダッシュボードとコスト上限設定
```

第3章の自己修復ループが本書の核心で、「CI赤 → Claudeが原因特定 → 修正コミット → CI再実行 → 緑」が全自動で回る。この構成を試し読みで体感したなら、続きは次章で待っている。
