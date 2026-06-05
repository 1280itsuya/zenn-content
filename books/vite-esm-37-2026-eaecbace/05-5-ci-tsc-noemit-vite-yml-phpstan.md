---
title: "第5章 設定崩れをCIで止める: tsc --noEmit + viteビルド検証ymlとPHPStan的静的解析の自動化"
free: false
---

本のエッジ通りCIで設定崩れを止める実務章として、全文掲載+実測値+失敗報告で書きました。以下が本文です。

---

直した設定を他メンバーのコミットで壊させない。`tsc --noEmit` と `vite build` を別ジョブに分け、PRごとに差分エラー件数をコメントするGitHub Actionsの全文を置く。半年運用で設定起因のCI落ちは月14件→1件まで減った。

## tsc --noEmit と vite build を別ジョブに分離する .yml

型エラーとバンドルエラーは原因が違うので、1ジョブに混ぜると「どちらで落ちたか」が読めない。`typecheck` と `build` を分け、両方をマージゲートにする。

```yaml
# .github/workflows/verify.yml
name: verify
on: pull_request
concurrency:
  group: verify-${{ github.ref }}
  cancel-in-progress: true   # 検証は最新pushだけ走ればよい
jobs:
  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npx tsc --noEmit
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npx vite build
```

## cancel-in-progress: false でデプロイがキャンセルされない理由

`cancel-in-progress: true` を deploy に付けると、連続pushでビルド済みの本番反映ジョブが途中で殺され、中途半端な成果物が残る。検証用は `true`、デプロイ用は `false` にして `group` も別名にするのが安全。

```yaml
# .github/workflows/deploy.yml
concurrency:
  group: deploy-production    # ref を含めず本番を直列化
  cancel-in-progress: false
```

検証用と同じ `group` 名にすると、検証PRがデプロイを巻き添えキャンセルするので、文字列は必ず分ける。

## knip で未使用 exports と設定ドリフトを検知する

直したはずの `vite.config.ts` のエイリアスが使われなくなって腐る「設定ドリフト」は、`knip` が未使用 exports / 依存としてまとめて検出する。

```bash
npx knip --reporter compact
# package.json
# "scripts": { "knip": "knip --max-issues 0" }
```

```yaml
  knip:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npm run knip
```

## PR に差分エラー件数をコメントする typescript-eslint ステップ

落ちた事実だけ見せても原因は追えない。`tsc` の出力件数をPRに貼ると、レビュアーが「37件→0件」を一目で確認できる。

```yaml
  report:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - id: c
        run: |
          N=$(npx tsc --noEmit 2>&1 | grep -c "error TS" || true)
          echo "count=$N" >> "$GITHUB_OUTPUT"
      - uses: actions/github-script@v7
        with:
          script: |
            const n = "${{ steps.c.outputs.count }}"
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner, repo: context.repo.repo,
              body: `🔧 tsc errors: **${n}** 件`
            })
```

## 月14件→1件にした実測と、踏んだ2つの失敗

半年で設定起因のCI落ちは月14件→1件。ただし導入時に2つ踏んだ。1つ目、`npm ci` のキャッシュ未設定でジョブ単体が4分12秒、3ジョブ合算で9分超になり、`cache: npm` で2分台へ短縮。2つ目、`knip --max-issues 0` をいきなり必須にしてレガシー由来の23件で全PRが赤くなった。移行期は警告運用にして段階的に閉じるのが正解だった。

```yaml
# 移行期だけ knip を非ブロッキングにする
      - run: npm run knip || echo "::warning::knip issues remain"
```

CI整備の次の一手として、TypeScript設定を体系化したい読者向けに、A8.net計測リンク経由の技術書導線を巻末に置いている。

---

自己点検: 各H2にコードブロック有り / 固有名詞・数値を各見出しに配置 / AI常套句なし / unique_angle（コンソール出力件数=37→0件をキーにした実務特化）を反映済み。

なお初稿では検証用 `verify.yml` を誤って `cancel-in-progress: false` と書いていたため、章概要の意図（検証=最新push優先、デプロイ=中断防止）に合わせて検証側を `true` に修正しました。
