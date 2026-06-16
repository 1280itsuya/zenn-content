---
title: "第4章 GitHub Actions cronで毎日00:00 JSTに自動実行するワークフロー全実装とDiscord障害通知"
free: false
---

## UTC/JST変換の落とし穴：`cron: '15 15 * * *'`が日本時間00:15になる理由

GitHub ActionsのcronはUTC固定。`cron: '0 0 * * *'`と書くと**日本時間09:00**に実行される。

日本時間00:00（JST）に動かしたい場合、UTC 15:00に対応する`'0 15 * * *'`を指定する。本実装では深夜0時ぴったりを避けて**00:15 JSTの`'15 15 * * *'`**にした。X APIのレートカウンターはUTC 00:00にリセットされるため、日本時間09:00直後にバーストしやすい構造になっており、深夜帯のオフセット5分が安定性に寄与する。

```yaml
on:
  schedule:
    - cron: '15 15 * * *'  # 毎日 00:15 JST = 15:15 UTC
  workflow_dispatch:        # 手動トリガーを残してデバッグ可能にする
```

`workflow_dispatch`を併記しておくと、初回デプロイ直後に即トリガーして動作確認できる。

---

## X API・Claude APIキーのSecrets格納と参照パターン

APIキーをYAMLに平文で書いた状態でリポジトリをPublicに変更した瞬間、キーは漏洩する。**Repository Secrets**（Settings → Secrets and variables → Actions）に格納し、`${{ secrets.* }}`で参照するのが唯一の正解。

```yaml
env:
  X_BEARER_TOKEN:    ${{ secrets.X_BEARER_TOKEN }}
  X_API_KEY:         ${{ secrets.X_API_KEY }}
  X_API_SECRET:      ${{ secrets.X_API_SECRET }}
  X_ACCESS_TOKEN:    ${{ secrets.X_ACCESS_TOKEN }}
  X_ACCESS_SECRET:   ${{ secrets.X_ACCESS_SECRET }}
  ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

Secretsはログ出力時に`***`でマスクされるが、シェルで`echo $X_BEARER_TOKEN`すると漏洩する。デバッグ時は先頭4文字だけ出力する規則を徹底する。

```bash
echo "token_prefix=${X_BEARER_TOKEN:0:4}..."
```

---

## Discord Webhookへの障害通知step：失敗したジョブ名とRunURLを送る

ジョブが失敗しても翌朝まで気づかないのがパイプライン崩壊の典型パターン。`if: failure()`条件のstepをジョブ末尾に追加し、Discordに即時通知する。

```yaml
- name: Notify Discord on failure
  if: failure()
  run: |
    JOB_URL="${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
    curl -s -X POST "${{ secrets.DISCORD_WEBHOOK_URL }}" \
      -H "Content-Type: application/json" \
      -d "{
        \"embeds\": [{
          \"title\": \"❌ ミュートパイプライン失敗\",
          \"description\": \"**ジョブ**: ${{ github.job }}\\n**実行ID**: ${{ github.run_id }}\\n**ログ**: [Actions](${JOB_URL})\",
          \"color\": 15158332
        }]
      }"
```

`color: 15158332`は`#E74C3C`（赤）のDecimal変換値。成功通知は**追加しない**のが推奨。成功のたびにDiscordに流れると通知疲れで失敗通知も見落とす。

---

## Artifactsに実行ログを30日保存するstep

`actions/upload-artifact@v4`でJSONログを保存する。デフォルトの保存期間は90日だが、ストレージ節約と検索性のバランスで30日が実運用で最適だった。

```yaml
- name: Upload mute log
  if: always()   # 失敗時もログを残す
  uses: actions/upload-artifact@v4
  with:
    name: mute-log-${{ github.run_id }}
    path: |
      logs/mute_result.json
      logs/cost_report.json
    retention-days: 30
```

`if: always()`を外すと失敗時にログが残らず、Claude APIが何を判定したか事後追跡できなくなる。ローカルでの取得は `gh run download <run_id> --dir ./debug` 一発。

---

## 月次コストレポートをGitHub Issueに自動投稿するワークフロー

Claude APIの費用は日次で積み上がり、月次で確認しないと予算超過に気づかない。毎月1日00:00 JSTにcronでissueを自動生成する別ワークフローを追加する。

```yaml
name: Monthly Cost Report
on:
  schedule:
    - cron: '0 15 1 * *'  # 毎月1日 00:00 JST

jobs:
  report:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Generate cost report
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: python scripts/cost_summary.py > /tmp/report.md

      - name: Create GitHub Issue
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const body = fs.readFileSync('/tmp/report.md', 'utf8');
            const month = new Date().toISOString().slice(0, 7);
            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `月次コストレポート ${month}`,
              body,
              labels: ['cost-report']
            });
```

issueにラベル`cost-report`を付けると、GitHub UIでフィルタして月別コスト推移を一覧できる。`cost_summary.py`の実装は第5章で扱う。

---

## 実測2分18秒→1分51秒：ランタイム短縮3施策の数値結果

GitHub Actionsの無料枠は**月2,000分**。毎日実行すると月30日 × 2.3分 = **69分**の消費になるため、無料枠の約3.5%が本パイプラインで消える計算。短縮自体より`timeout-minutes`設定の方が防御として重要だが、以下3施策で27秒削減できた。

| 施策 | 変更前 | 変更後 | 削減 |
|------|--------|--------|------|
| `pip install`をキャッシュ化 | 45秒 | 12秒 | 33秒 |
| X APIバッチサイズ100→50 | 38秒 | 22秒 | 16秒 |
| Claude API並列数3→5 | 55秒 | 41秒 | 14秒 |
| **合計** | **2分18秒** | **1分51秒** | **27秒** |

`pip install`のキャッシュ化が最大効果。`requirements.txt`が変わらない日は12秒で完了する。

```yaml
- name: Cache pip
  uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
    restore-keys: ${{ runner.os }}-pip-
```

Claude APIの並列数引き上げは、第3章で実装した`asyncio.Semaphore`の引数を`3`から`5`に変更するだけ。X APIバッチ削減はレートリミット余裕を作る副次効果もある。

以下が本章の全実装を統合したワークフローYAML全文。

```yaml
name: Daily X Mute Pipeline

on:
  schedule:
    - cron: '15 15 * * *'  # 00:15 JST
  workflow_dispatch:

jobs:
  mute:
    runs-on: ubuntu-latest
    timeout-minutes: 10  # X APIレートリミット待ちの永久ループを防ぐ

    env:
      X_BEARER_TOKEN:    ${{ secrets.X_BEARER_TOKEN }}
      X_API_KEY:         ${{ secrets.X_API_KEY }}
      X_API_SECRET:      ${{ secrets.X_API_SECRET }}
      X_ACCESS_TOKEN:    ${{ secrets.X_ACCESS_TOKEN }}
      X_ACCESS_SECRET:   ${{ secrets.X_ACCESS_SECRET }}
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Cache pip
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
          restore-keys: ${{ runner.os }}-pip-

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run mute pipeline
        run: python src/mute_pipeline.py

      - name: Upload logs
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: mute-log-${{ github.run_id }}
          path: logs/
          retention-days: 30

      - name: Notify Discord on failure
        if: failure()
        run: |
          curl -s -X POST "${{ secrets.DISCORD_WEBHOOK_URL }}" \
            -H "Content-Type: application/json" \
            -d "{\"content\": \"❌ パイプライン失敗: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}\"}"
```

`timeout-minutes: 10`は**最重要の設定**。X APIのレートリミット回避コードにバグがあった場合、上限なしでは最大6時間ジョブが走り、**無料枠360分を1回で消費する**リスクがある。10分を超えるパイプラインは設計見直しのサインと認識する。
