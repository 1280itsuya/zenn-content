---
title: "第4章：GitHub Actionsで毎朝7時に全自動運用——CI/CDとスケジューラー設計の実際"
free: false
---

## GitHub Actions cronで毎朝7時に起動する

`.github/workflows/daily_pipeline.yml` を作成し、`schedule` トリガーで実行する。タイムゾーンはUTCなので、JST 7:00 = UTC 22:00（前日）になる点に注意。

```yaml
name: daily-content-pipeline

on:
  schedule:
    - cron: '0 22 * * *'  # JST 07:00
  workflow_dispatch:       # 手動実行用

jobs:
  run:
    runs-on: ubuntu-latest
    timeout-minutes: 30    # ハング防止

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - run: pip install -r requirements.txt

      - name: Run pipeline
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          ZENN_GITHUB_TOKEN: ${{ secrets.ZENN_GITHUB_TOKEN }}
          QIITA_TOKEN: ${{ secrets.QIITA_TOKEN }}
        run: python orchestrator/main.py
```

`workflow_dispatch` を併記しておくと、Actionsタブから即時テスト実行できる。開発中は毎日待てないため、これがないと検証コストが跳ね上がる。

---

## SecretsにAPIキーを安全に渡す

リポジトリの **Settings → Secrets and variables → Actions → New repository secret** から登録する。ローカルの `.env` に書いているキーを1つずつ移す。

```
ANTHROPIC_API_KEY    ← Claude API
ZENN_GITHUB_TOKEN    ← Zenn連携用のGitHub PAT
QIITA_TOKEN          ← Qiita API
```

コード側では `os.environ["ANTHROPIC_API_KEY"]` で読む。`python-dotenv` の `load_dotenv()` はActions環境では何もしないが、ローカル実行時は `.env` を読むため、両方の動作を壊さずに済む。

**落とし穴：** PAT（Personal Access Token）を `repo` スコープ全開で発行しない。Zenn連携なら `contents: write` のみで十分。最小権限が原則。

---

## 失敗時の通知を設計する

無人運転で一番困るのは「静かに失敗し続ける」こと。Actionsのデフォルト失敗通知（メール）は埋もれやすいため、Discordへのwebhook通知を追加する。

```yaml
      - name: Notify on failure
        if: failure()
        run: |
          curl -H "Content-Type: application/json" \
            -d "{\"content\": \"❌ pipeline失敗 $(date '+%Y-%m-%d') — ${{ github.run_url }}\"}" \
            ${{ secrets.DISCORD_WEBHOOK_URL }}
```

`if: failure()` で前ステップの失敗時のみ実行される。`github.run_url` を本文に入れておくと、Discordから直接ログへ飛べる。

---

## timeout-minutes を必ず設定する

Claude APIへのリクエストがタイムアウトせず、ジョブが6時間ハングし続けるケースが実際に発生する。`timeout-minutes: 30` を設定しておけば強制終了され、失敗通知が飛ぶ。設定しない場合、GitHub Actionsは最大6時間待ち続け、次の日のcronまで気づかない。

---

## ローカル→Actions の動作差異チェックリスト

| 確認点 | ローカル | Actions |
|---|---|---|
| `.env` の読み込み | `load_dotenv()` | Secretsで代替 |
| タイムゾーン | JST | UTC（変換必須） |
| ファイルパス | 絶対パス混在 | `os.path.join` 推奨 |
| インタラクティブ入力 | 可 | 不可（必ずハング）|

`git push` 時のGitHub認証ダイアログも無人環境ではTTY待ちでハングする。`GIT_TERMINAL_PROMPT=0` を環境変数に追加し、PAT埋め込みURLでpushする設計にしておく。

```python
repo_url = f"https://{token}@github.com/{org}/{repo}.git"
subprocess.run(["git", "push", repo_url, "main"],
               env={**os.environ, "GIT_TERMINAL_PROMPT": "0"},
               check=True)
```

これで毎朝7時に無人で記事生成・投稿・通知まで完走するパイプラインが動き続ける。
