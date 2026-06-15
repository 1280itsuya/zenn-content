---
title: "GitHub ActionsでClaude生成ジョブをCI/CDに組み込む"
free: false
---

## GitHub ActionsでClaude生成ジョブをCI/CDに組み込む

### なぜGitHub Actionsなのか

ローカルのスクリプトをcronで回すと、PCがスリープしただけで止まる。VPSを借りれば維持費がかかる。GitHub Actionsなら**パブリックリポジトリは無料・月2,000分**のランナーが使え、シークレット管理もUIから完結する。記事生成のような軽量ジョブには過剰スペックだが、信頼性とゼロ運用コストを両立できる点で個人開発の最適解だ。

---

## ワークフロー全体像

```
.github/workflows/
└── daily_generate.yml   ← 毎朝7時に起動
src/
├── generate.py          ← Claude APIで記事生成
└── poster.py            ← Zenn/Qiitaへ投稿
```

ジョブの流れは「生成 → 差分コミット → 投稿 → Slack通知」の4ステップ。

---

## ワークフローファイルの全文

```yaml
name: daily-generate

on:
  schedule:
    - cron: '0 22 * * *'   # UTC 22:00 = JST 07:00
  workflow_dispatch:         # 手動実行ボタン

jobs:
  generate:
    runs-on: ubuntu-latest
    permissions:
      contents: write        # git push に必要

    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GH_PAT }}   # fine-grained PATを使う

      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Generate articles
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          ZENN_TOKEN: ${{ secrets.ZENN_TOKEN }}
        run: python src/generate.py

      - name: Commit generated files
        run: |
          git config user.name  "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add data/articles/
          git diff --cached --quiet || git commit -m "chore: auto-generate articles $(date -u +%Y-%m-%d)"
          git push

      - name: Post to platforms
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          ZENN_TOKEN: ${{ secrets.ZENN_TOKEN }}
          QIITA_TOKEN: ${{ secrets.QIITA_TOKEN }}
        run: python src/poster.py

      - name: Slack notification
        if: always()
        uses: slackapi/slack-github-action@v1.27.0
        with:
          payload: |
            {
              "text": "${{ job.status == 'success' && '✅' || '❌' }} daily-generate: ${{ job.status }}\n${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## シークレットの登録

**Settings → Secrets and variables → Actions → New repository secret** で以下を登録する。

| シークレット名 | 値 |
|---|---|
| `ANTHROPIC_API_KEY` | Anthropicコンソールで発行したキー |
| `GH_PAT` | `contents: write` 権限のfine-grained PAT |
| `ZENN_TOKEN` / `QIITA_TOKEN` | 各プラットフォームのAPIトークン |
| `SLACK_WEBHOOK_URL` | Slack Incoming Webhookの URL |

`GH_PAT` に `GITHUB_TOKEN` を使いたくなるが、**デフォルトの `GITHUB_TOKEN` でpushするとActionsが再トリガーされないため、fine-grainedのPATが必須**。

---

## 落とし穴と対処

**① タイムゾーンのズレ**
`cron` はUTC固定。「JST 7:00 = UTC 22:00（前日）」なので日付をまたぐことに注意。`date -u +%Y-%m-%d` で常にUTC日付を使うか、`TZ=Asia/Tokyo date` を明示する。

**② 差分なしでのコミット失敗**
生成結果が前日と同じだと `git commit` が終了コード1を返してジョブが失敗する。

```bash
git diff --cached --quiet || git commit -m "..."
```

`||` で「差分がなければスキップ」にしておくだけで解決する。

**③ APIキーのログ漏洩**
`run: python src/generate.py` の出力にキーが混入するケースがある。生成スクリプト内で `logging` を使う場合は `ANTHROPIC_API_KEY` をログに含めないよう `logging.getLogger("httpx").setLevel(logging.WARNING)` で抑制する。

**④ ランナーの並列実行**
`workflow_dispatch` で手動実行すると定期ジョブと重なることがある。

```yaml
concurrency:
  group: daily-generate
  cancel-in-progress: false
```

`cancel-in-progress: false` で先行ジョブが終わるまで後続をキューに積む。`true` にすると先行ジョブが強制終了されて記事が中途半端な状態でコミットされる危険がある。

---

## 動作確認の手順

1. ワークフローを `main` にpushする
2. **Actions タブ → Run workflow** で手動実行
3. ログの `Commit generated files` ステップで `git push` が成功しているか確認
4. リポジトリの `data/articles/` に当日付きのファイルが増えていれば完了
5. Slackに緑の✅が届いたら本番運用に移行する

---

手動実行ボタン（`workflow_dispatch`）を常に残しておくと、生成ロジックを変えたときの即時テストに使えて便利だ。ワークフローが安定したら `on.schedule` だけ残してシンプルにする必要はない。「手動＋定期」の両立が、個人開発の運用コストを最小に保つ。
