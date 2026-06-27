---
title: "GitHub Actions + tsc --noEmitでESMエラーをPR単位で事前検知するCIテンプレート完全版"
free: false
---

## tsc --noEmitをpush時に実行する最小ワークフロー（実測18秒）

`.github/workflows/type-check.yml` を以下の内容で作成する。これがすべての基点になる。

```yaml
name: type-check

on:
  push:
    branches: ['**']
  pull_request:
    branches: [main]

jobs:
  tsc:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npx tsc --noEmit
```

`ubuntu-latest` + Node 20 + `npm ci` の構成でTypeScript 5.5の型チェックが実測18秒。`npm install`ではなく`npm ci`にするだけで、package-lock.jsonを厳密に使い再現性が上がる。

## node_modules + .tsbuildinfo をキャッシュしてCI時間を35%削減する

キャッシュなしだと毎回`npm ci`に12〜15秒かかる。`actions/cache`でnode_modulesと`.tsbuildinfo`（インクリメンタルビルド情報）を分離してキャッシュすると合計時間が約12秒に短縮される（35%削減）。

```yaml
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: cache node_modules
        id: cache-npm
        uses: actions/cache@v4
        with:
          path: node_modules
          key: npm-${{ runner.os }}-${{ hashFiles('package-lock.json') }}

      - run: npm ci
        if: steps.cache-npm.outputs.cache-hit != 'true'

      - name: cache .tsbuildinfo
        uses: actions/cache@v4
        with:
          path: .tsbuildinfo
          key: tsbuild-${{ runner.os }}-${{ github.sha }}
          restore-keys: |
            tsbuild-${{ runner.os }}-

      - run: npx tsc --noEmit --incremental
```

## Viteビルドキャッシュと型チェックキャッシュを分離する理由と実測値

Viteは`.vite/cache`（ブラウザ向けトランスパイル結果）を、tscは`.tsbuildinfo`（型情報のみ）を別々に管理する。両者を同一キャッシュキーで混在させると、Viteキャッシュのバスト（依存関係更新時）のたびに`.tsbuildinfo`も消え、型チェックがフルスキャンに戻る。分離することで型修正だけのCI再実行がViteの再ビルドをスキップし、実測で4〜6秒の短縮になる。

```yaml
      # Viteビルドキャッシュ（build jobと共有する場合のみ追加）
      - name: cache vite
        uses: actions/cache@v4
        with:
          path: .vite
          key: vite-${{ runner.os }}-${{ hashFiles('vite.config.*') }}-${{ github.sha }}
          restore-keys: |
            vite-${{ runner.os }}-${{ hashFiles('vite.config.*') }}-
            vite-${{ runner.os }}-
```

型チェックジョブとViteビルドジョブを独立したjobに分け、それぞれのキャッシュキーを持たせることが前提。同一jobに両方まとめると依存関係が生まれ、キャッシュ分離の効果が薄れる。

## Claude Sonnet 4.6でエラー診断しPRコメントへ自動投稿するアクション設定

第4章のClaude診断スクリプト（`scripts/diagnose.py`）をCI上で実行し、tscエラーをPRコメントに自動投稿する。あらかじめ`ANTHROPIC_API_KEY`をリポジトリのSecrets（Settings → Secrets → Actions）に登録してから、以下のステップを`tsc`ジョブの末尾に追加する。

```yaml
      - name: run tsc and capture errors
        id: tsc-run
        run: |
          npx tsc --noEmit 2>&1 | tee /tmp/tsc-errors.txt || true

      - name: claude diagnosis
        if: github.event_name == 'pull_request'
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          pip install anthropic --quiet
          python scripts/diagnose.py --input /tmp/tsc-errors.txt --output /tmp/suggestion.md

      - name: post PR comment
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs')
            const body = fs.readFileSync('/tmp/suggestion.md', 'utf8')
            if (!body.trim()) return
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Claude診断結果\n\n${body}`
            })
```

tscエラーがゼロなら`/tmp/tsc-errors.txt`が空になり、`diagnose.py`側でスキップ処理を入れておくとAPI費用が発生しない。エラーがある場合のみClaude Sonnet 4.6を呼び出し、1回あたり¥2〜5で`tsconfig.json`の差分提案がPRコメントに貼られる。

## GitHub Actions無料枠2,000分/月を超えない設定と月次試算

privateリポジトリは月2,000分が上限（publicは無制限）。型チェックジョブが12秒なら、月166回のプッシュで約33分の消費。1人開発では現実的に超過しないが、`paths`フィルタを加えると実行回数をさらに30〜40%削減できる。

```yaml
on:
  push:
    branches: ['**']
    paths:
      - '**/*.ts'
      - '**/*.tsx'
      - 'tsconfig*.json'
      - 'package-lock.json'
  pull_request:
    branches: [main]
    paths:
      - '**/*.ts'
      - '**/*.tsx'
      - 'tsconfig*.json'
      - 'package-lock.json'
```

`.md`・`.css`・画像ファイルの変更では型チェックが走らない。Claude診断ステップは`if: github.event_name == 'pull_request'`で囲んでいるため、単純なpushではAPI費用が発生しない。月100PRで試算しても診断コストは最大¥500、Actions分数は20分以内に収まる計算になる。
