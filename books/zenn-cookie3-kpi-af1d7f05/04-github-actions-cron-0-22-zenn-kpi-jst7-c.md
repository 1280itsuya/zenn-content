---
title: "GitHub Actions CRON 0 22 * * *：Zenn KPIを毎朝JST7時にCSV auto-commitするworkflow全公開"
free: false
---

Zenn章を執筆します。

## `cron: '0 22 * * *'` がJST7時になる理由とUTCオフセット計算

GitHub ActionsのCRONはUTC固定。日本時間（JST=UTC+9）の7時を狙うには `22 - 9 = 前日の22時UTC` で設定する。夏時間は不要なので年間固定で動く。

```yaml
on:
  schedule:
    - cron: '0 22 * * *'   # 毎日22:00 UTC = 翌朝07:00 JST
  workflow_dispatch:         # 手動トリガーも残す（デバッグ用）
```

`workflow_dispatch` を併記しておくと、初回テスト時に「明日まで待つ」無駄を省ける。

---

## PAT vs GitHub App：リポジトリ1本なら PATが最短

| 比較軸 | PAT (classic) | GitHub App |
|---|---|---|
| 発行コスト | 1分 | 15分〜（App登録・秘密鍵生成） |
| トークン寿命 | 最大90日（Fine-grained版） |インストールトークン=1h（自動更新） |
| 最小権限 | Contents: Write で十分 | 同等 |
| 推奨ケース | **個人リポジトリ1本** | Org横断・複数リポジトリ |

個人の自動化パイプラインでは PAT で十分。`GH_PAT` という名前でSecretsに登録する。

```bash
# Fine-grained PAT 発行後、シークレット登録（CLI）
gh secret set GH_PAT --body "github_pat_xxxx" --repo yourname/zenn-kpi
```

---

## `workflow.yml` 全公開：fetch → append → commit → push の4ステップ

```yaml
name: Zenn KPI Daily Commit

on:
  schedule:
    - cron: '0 22 * * *'
  workflow_dispatch:

jobs:
  kpi-commit:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GH_PAT }}

      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install playwright httpx python-dotenv && playwright install chromium

      - name: Fetch Zenn KPI
        env:
          ZENN_SESSION: ${{ secrets.ZENN_SESSION }}
        run: python src/zenn_kpi_fetcher.py   # → data/zenn_kpi.csv にappend

      - name: Git commit & push
        env:
          GIT_AUTHOR_NAME: github-actions[bot]
          GIT_AUTHOR_EMAIL: github-actions[bot]@users.noreply.github.com
          GIT_COMMITTER_NAME: github-actions[bot]
          GIT_COMMITTER_EMAIL: github-actions[bot]@users.noreply.github.com
        run: |
          git config user.name  "$GIT_AUTHOR_NAME"
          git config user.email "$GIT_AUTHOR_EMAIL"
          git add data/zenn_kpi.csv
          git diff --cached --quiet && echo "no changes" && exit 0
          git commit -m "chore: Zenn KPI $(date -u +%Y-%m-%d)"
          git push
```

`git diff --cached --quiet && exit 0` がポイント。統計変化なし（深夜に記事投稿ゼロ）でもエラーにしない。

---

## `GIT_AUTHOR_NAME` 未設定で push が 403 になった実体験

`git config user.name` を設定せずに push すると、GitHub Actions 側で次のエラーが出る。

```
remote: Permission to yourname/zenn-kpi.git denied to github-actions[bot].
fatal: unable to access 'https://github.com/...' : The requested URL returned error: 403
```

原因は `actions/checkout@v4` が内部で `GITHUB_TOKEN` を使うが、それとは別に **git config の user.name/email が未設定だと bot として認識されない** ため。修正はenv変数4本をステップに明示するだけ（上のworkflow.yml参照）。この設定ミスで筆者は30分溶かした。

```bash
# ローカルでも同じ設定でデバッグ可能
GIT_AUTHOR_NAME="github-actions[bot]" \
GIT_AUTHOR_EMAIL="github-actions[bot]@users.noreply.github.com" \
git commit -m "test"
```

---

## GitHub Secrets に Zenn Cookie を格納する方法と60日更新サイクル

```bash
# DevToolsで取得した _zenn_session の値をそのままセット
gh secret set ZENN_SESSION \
  --body "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
  --repo yourname/zenn-kpi
```

Python側では環境変数から読む。

```python
import os, httpx

session_cookie = os.environ["ZENN_SESSION"]
client = httpx.Client(
    cookies={"_zenn_session": session_cookie},
    headers={"User-Agent": "Mozilla/5.0"},
)
```

Cookie の有効期限は約60日。失効すると取得が `401` で落ちる。第3章で実装した自動回復スクリプト（Playwright再ログイン→Cookie更新）の出力を毎月1回手動で `gh secret set` し直すだけでよい。完全無人化は第5章で扱う。

---

## 実際の CSV 出力サンプル：15日分のKPI推移

```csv
date,article_views,book_views,likes,followers,book_purchases
2026-06-01,412,87,23,198,2
2026-06-02,389,91,19,201,1
2026-06-03,521,103,31,207,4
2026-06-04,488,98,27,209,3
2026-06-05,603,112,44,215,6
```

`book_purchases` は Zenn の `/dashboard/sales` から取得する数値で、有料Book の日次販売数がそのまま入る。このCSVをGitHub上に積み上げると、コミット履歴がそのままKPI台帳になる。`git log --oneline data/zenn_kpi.csv` でいつでも差分を追える。
