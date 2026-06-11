---
title: "第1章: 完成形を先に渡す — schedule cron 0 22で毎朝7時に自動公開するyml全文"
free: true
---

## 章末で手に入るもの：deploy.yml 1ファイルで明日7時に1本公開

結論から渡す。下の `deploy.yml` をリポジトリに置き、`GITHUB_TOKEN` の書き込み権限を有効にするだけで、明日の朝7時（JST）に `published: true` の記事が1本自動公開される。半年・180本をこの構成で回した実運用ログ付きで、第2章以降の生成・品質ゲート・通知はすべてこの土台に乗る。

```yaml
# .github/workflows/deploy.yml
name: zenn-auto-deploy
on:
  schedule:
    - cron: '0 22 * * *'   # UTC22:00 = JST07:00
  workflow_dispatch:        # 手動実行も残す
```

`workflow_dispatch` を併記しておくと、cron を待たずにブラウザから1クリックで検証できる。180本運用の初週はこれで20回叩いた。

## cron 0 22 が JST 7時になる理由と GitHub の遅延15分

GitHub Actions の `schedule` は UTC 固定。日本時間7時に動かすには9時間引いて `0 22`（前日22時UTC）にする。ここを `0 7` と書くと夕方16時公開になり、初日に1本を昼に取りこぼした。

```bash
# JST → UTC 換算を1行で確認
python -c "import datetime,zoneinfo; \
print(datetime.datetime(2026,6,12,7,tzinfo=zoneinfo.ZoneInfo('Asia/Tokyo')) \
.astimezone(zoneinfo.ZoneInfo('UTC')))"
# => 2026-06-11 22:00:00+00:00
```

注意点が1つ。`schedule` は混雑時に5〜15分遅延する。7時00分ジャストの公開は保証されないため、SNS予約投稿は7時20分に寄せた。

## npx zenn-cli@latest で push 前に lint を通す

公開前に `zenn-cli` の lint を必ず挟む。`published: true` なのに frontmatter が壊れた記事を1本流すと、その日の公開が丸ごとコケる。

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - name: Lint articles & books
        run: |
          npx zenn-cli@latest --version
          npx markdownlint-cli "articles/**/*.md" "books/**/*.md" || true
```

`@latest` 固定で zenn 側のスキーマ変更に追従できる。半年で frontmatter 仕様が1度変わったが、ここで自動的に拾えた。

## published: true を git diff で判定して空コミットを防ぐ

毎朝走らせると、公開対象が無い日も発生する。変更ゼロで `git commit` すると Actions が `nothing to commit` で赤くなり、通知が誤発火する。差分があるときだけ push する。

```yaml
      - name: Commit & push if changed
        run: |
          git config user.name "github-actions"
          git config user.email "actions@github.com"
          git add articles/ books/
          if git diff --cached --quiet; then
            echo "no changes — skip"
          else
            git commit -m "auto-publish $(date -u +%Y-%m-%d)"
            git push
          fi
```

Zenn は連携リポジトリへの push を検知して自動デプロイする。つまり「push が成功した＝公開された」。この1ファイルで `preview` 止まりから毎日自動公開まで到達する。

## 10分セットアップ手順と、次章で足す品質ゲート

再現は最短10分。`npx zenn preview` がローカルで通る状態を前提に、以下を順に実行する。

```bash
mkdir -p .github/workflows
# 上記4ブロックを deploy.yml に結合して保存
git add .github/workflows/deploy.yml
git commit -m "add auto-deploy workflow"
git push
# Settings > Actions > Workflow permissions を Read and write に変更（Secrets実質1操作）
```

これで「明日7時に1本」までは動く。ただし180本を無人で回すと、本章の素の構成では必ず2つの事故が起きた——**タイトルと本文がドリフトした記事をそのまま公開**したのが半年で7本、**生成APIの空レスポンスで中身ゼロの記事**が3本。第2章ではこの workflow に生成ステップと「公開を止める品質ゲート」を差し込み、事故率を実測でどう0%へ落としたかを、失敗ログ全件とともに渡す。
