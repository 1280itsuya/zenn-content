---
title: "第3章：GitHub Actions cron×Secretsでミュート処理を毎朝6時に完全無人化するYAML全文"
free: false
---

## `cron: '0 21 * * *'` ―― JST 6:00 起動の時差設計

GitHub Actions の `cron` はUTC固定。JST 6:00 = UTC 21:00 なので `0 21 * * *` と書く。設定ミスの確認は `gh run list` で実行履歴を引くのが最速だ。

```bash
gh run list --workflow=mute.yml --limit 5 --json startedAt,status \
  | jq '.[] | "\(.startedAt) \(.status)"'
```

初回デプロイ後は必ず `workflow_dispatch` も定義して手動トリガーし、タイムゾーン起因のズレを当日中に潰す。

---

## Secrets に X\_BEARER\_TOKEN と ANTHROPIC\_API\_KEY を登録する3ステップ

Settings → Secrets and variables → Actions → New repository secret。登録後はYAML内で `${{ secrets.X_BEARER_TOKEN }}` と参照する。ハードコード厳禁。

```yaml
env:
  X_BEARER_TOKEN: ${{ secrets.X_BEARER_TOKEN }}
  ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

CLIからは `gh secret set X_BEARER_TOKEN < token.txt` でバルク登録もできる。`.env` ファイルをリポジトリに含めると即BANなので、`.gitignore` への追加も合わせて確認する。

---

## ミュート済みIDを JSONストアに永続化する actions/cache 設計

同じアカウントを翌日再ミュートしないために `muted_ids.json` をキャッシュする。キャッシュキーは日付を含めず固定にして、ジョブをまたいで上書き保存する構成にする。

```yaml
- name: Restore muted ID cache
  uses: actions/cache@v4
  with:
    path: data/muted_ids.json
    key: muted-ids-store
    restore-keys: muted-ids-store

- name: Save muted ID cache
  if: always()
  uses: actions/cache/save@v4
  with:
    path: data/muted_ids.json
    key: muted-ids-store
```

`restore-keys` をキー本体と同一にすると、初回キャッシュなしでもスキップせず続行する。

---

## 実行ログを artifacts に保存して誤爆ログを蓄積する

誤爆削減には過去の判断履歴が必要。`upload-artifact` で90日間保持し、第4章のプロンプト改善ループの入力に使う。

```yaml
- name: Upload mute log
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: mute-log-${{ github.run_id }}
    path: logs/mute_run.jsonl
    retention-days: 90
```

`mute_run.jsonl` は1行1件の構造で記録する。

```jsonl
{"ts": "2026-06-29T21:03:12Z", "user_id": "9876543210", "reason": "スパムDM連投", "action": "muted"}
{"ts": "2026-06-29T21:03:15Z", "user_id": "1234567890", "reason": "フォロワー数条件外", "action": "skipped"}
```

このJSONLが第4章でClaude APIに渡す「誤爆事例コーパス」になる。

---

## 無料プラン2,000分/月に収める pip キャッシュチューニング

`pip install` を毎回走らせると約90秒かかる。`actions/cache` でキャッシュすると15秒まで短縮でき、1実行あたり約2分10秒に収まる。月30回実行で65分消費 ―― 2,000分制限の3.3%だ。

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: pip-${{ hashFiles('requirements.txt') }}

- run: pip install -r requirements.txt
```

`requirements.txt` が変わらない限り2回目以降はキャッシュヒットする。

---

## 本番稼働YAML全文（mute.yml）―― 3ヶ月無停止で動いた設定そのまま

```yaml
name: Auto Mute Bot

on:
  schedule:
    - cron: '0 21 * * *'   # JST 06:00
  workflow_dispatch:

permissions:
  contents: read

jobs:
  mute:
    runs-on: ubuntu-latest
    timeout-minutes: 10   # API障害でハングして2000分を使い切るのを防ぐ

    env:
      X_BEARER_TOKEN: ${{ secrets.X_BEARER_TOKEN }}
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
          key: pip-${{ hashFiles('requirements.txt') }}

      - run: pip install -r requirements.txt

      - name: Restore muted ID store
        uses: actions/cache@v4
        with:
          path: data/muted_ids.json
          key: muted-ids-store
          restore-keys: muted-ids-store

      - name: Run mute pipeline
        run: python src/mute_pipeline.py
        # exit 1 = ミュート0件, exit 2 = API障害。両方をSlack通知対象にする

      - name: Save muted ID store
        if: always()
        uses: actions/cache/save@v4
        with:
          path: data/muted_ids.json
          key: muted-ids-store

      - name: Upload run log
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: mute-log-${{ github.run_id }}
          path: logs/mute_run.jsonl
          retention-days: 90
```

`timeout-minutes: 10` は省略不可。3ヶ月の実測では最長実行が4分17秒。制限の半分以下で安定している。`if: always()` をキャッシュ保存とログアップロードの両方に付けておかないと、パイプラインが途中失敗した日にIDストアが消えて翌日から再ミュートが発生する ―― この設計ミスで最初の2週間、誤爆率が下がらなかった原因の一つだった。
