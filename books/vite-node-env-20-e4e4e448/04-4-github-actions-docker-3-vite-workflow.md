---
title: "第4章 GitHub Actions / Docker × シークレット注入3パターン — VITE_ プレフィックス漏洩ゼロ設計と workflow YAML 完全版"
free: false
---

## パターン別採点表: `env:` 直書き / `.env` 生成 / OIDC+Vault の3択

3パターンを「漏洩リスク・運用コスト・CI速度」の3軸で採点する。先に結論を示す。

| パターン | 漏洩リスク | 運用コスト | CI速度 | 合計/9 |
|---|---|---|---|---|
| 1. `env:` ブロック直書き | △ 3 | ◯ 1 | ◯ 1 | **5** |
| 2. `.env` ファイル生成 | ○ 2 | △ 2 | ○ 1 | **5** |
| 3. OIDC + Vault + Environments | ◎ 1 | △ 3 | △ 2 | **6** |

数値は小さいほど優秀。このあと各パターンのコードを示しながら加点・減点の根拠を説明する。

---

## VITE_ プレフィックスが bundle に焼き込まれる仕組みと漏洩リスクの定量

Vite は **ビルド時** に `import.meta.env.VITE_*` を文字列リテラルで置換する。つまり `VITE_API_KEY=sk-live-xxxxxx` を CI の環境変数として渡すと、生成された `dist/assets/index-XXXXXXXX.js` の中に平文で埋め込まれる。

```bash
# ビルド後の bundle に VITE_ 変数が残っているか確認するワンライナー
grep -rE 'VITE_[A-Z_]+\s*=\s*[^$]' dist/ && echo "⚠️  SECRET LEAKED" || echo "✅ clean"
```

GitHub Actions のデフォルト設定では `dist/` が artifact としてアップロードされることが多く、**Actionsの "Artifacts" タブから誰でもダウンロード可能** になる。2023年に報告されたケースでは、OSSリポジトリの fork PR が悪意ある `workflow_run` トリガーでシークレットを抜いた事例がある（[GitHub Blog, 2023-09](https://github.blog/security/supply-chain-security/)）。

**対策の鉄則**: `VITE_` をつける変数はパブリックな値のみ。APIキー・DBパスワード・OAuthシークレットは `VITE_` を絶対につけない。

---

## パターン1: `env:` ブロック直書き — 最速だが artifact 漏洩リスクあり

```yaml
# .github/workflows/deploy-pattern1.yml
name: Deploy (Pattern 1 - env block)

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      # ✅ パブリック設定値のみ VITE_ をつける
      VITE_APP_TITLE: ${{ vars.APP_TITLE }}
      VITE_API_BASE_URL: ${{ vars.API_BASE_URL }}
      # ❌ これが漏洩パターン — VITE_ をつけてはいけない
      # VITE_API_KEY: ${{ secrets.API_KEY }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run build

      # artifact アップロード前に bundle をスキャン
      - name: Leak scan
        run: |
          if grep -rqE '[A-Za-z0-9+/]{32,}' dist/assets/*.js; then
            echo "::warning::bundle に長い文字列があります。シークレット混入を確認してください"
          fi

      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/
          retention-days: 1  # 最短期間に絞る
```

`env:` ブロックはシンプルだが、**誤って `VITE_` をつけた秘匿値を渡すと即アウト**。後述するパターン3のゲートが最終防衛線になる。

---

## パターン2: `.env` ファイル生成ステップ — 書き捨てと消去漏れ対策

```yaml
# .github/workflows/deploy-pattern2.yml
name: Deploy (Pattern 2 - .env file generation)

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Generate .env (秘匿値は VITE_ なし、パブリック値のみ VITE_ 付き)
        run: |
          cat <<EOF > .env
          VITE_APP_TITLE=${{ vars.APP_TITLE }}
          VITE_API_BASE_URL=${{ vars.API_BASE_URL }}
          # サーバーサイド専用 — VITE_ なし
          DATABASE_URL=${{ secrets.DATABASE_URL }}
          INTERNAL_API_KEY=${{ secrets.INTERNAL_API_KEY }}
          EOF

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run build

      - name: 必ず .env を削除 (artifact に混入させない)
        if: always()  # build 失敗時も実行
        run: rm -f .env

      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/
```

`if: always()` を忘れると **ビルド失敗時に `.env` が残り、後続ステップで artifact に含まれる** ことがある。`always()` は必須。

---

## パターン3: OIDC + HashiCorp Vault + GitHub Environments 承認ゲート

本番デプロイには人間の承認を挟む。GitHub Environments の "Required reviewers" と OIDC 認証を組み合わせると、シークレットが CI ランナーに届くのは承認後のみになる。

```yaml
# .github/workflows/deploy-pattern3.yml
name: Deploy (Pattern 3 - OIDC + Vault)

on:
  push:
    branches: [main]

permissions:
  id-token: write   # OIDC トークン発行に必須
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci && npm run build

      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production   # ← ここで承認ゲートが発動
    steps:
      - name: OIDC で Vault に認証
        uses: hashicorp/vault-action@v3
        with:
          url: ${{ secrets.VAULT_ADDR }}
          method: jwt
          role: github-actions-deploy
          secrets: |
            secret/data/myapp DATABASE_URL | DATABASE_URL ;
            secret/data/myapp INTERNAL_API_KEY | INTERNAL_API_KEY

      - uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/

      - name: Deploy to server
        run: |
          # DATABASE_URL, INTERNAL_API_KEY は Vault から注入済み
          # dist/ には VITE_ パブリック値のみ焼き込み済み
          rsync -az dist/ user@${{ vars.DEPLOY_HOST }}:/var/www/html/
```

GitHub リポジトリの **Settings → Environments → production → Required reviewers** に承認者を設定する。`deploy` ジョブは承認されるまで `VAULT_ADDR` や Vault シークレットへのアクセスが遮断される。

---

## Docker Compose の ARG vs ENV 混乱を Dockerfile サンプルで解消

Vite をコンテナでビルドする場合、`ARG` と `ENV` の使い分けがトラブルの温床になる。

```dockerfile
# Dockerfile.build
FROM node:20-alpine AS builder

# ARG: ビルド時のみ有効、コンテナイメージの ENV レイヤーには残らない
ARG VITE_APP_TITLE
ARG VITE_API_BASE_URL

# ENV に昇格させると docker inspect でランタイムから見えてしまう — 禁止
# ENV VITE_API_BASE_URL=$VITE_API_BASE_URL  ← やってはいけない

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .

# ARG は npm run build (= vite build) の実行時に参照可能
RUN VITE_APP_TITLE=$VITE_APP_TITLE \
    VITE_API_BASE_URL=$VITE_API_BASE_URL \
    npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
```

```yaml
# docker-compose.yml
services:
  build:
    build:
      context: .
      dockerfile: Dockerfile.build
      args:
        VITE_APP_TITLE: ${VITE_APP_TITLE}
        VITE_API_BASE_URL: ${VITE_API_BASE_URL}
        # ❌ 秘匿値を ARG で渡すと docker history に平文で残る
        # INTERNAL_API_KEY: ${INTERNAL_API_KEY}
```

`docker history <image>` を実行すると `ARG` で渡した値がレイヤーメタデータとして**平文で見える**。`INTERNAL_API_KEY` のような秘匿値は ARG に渡さず、ランタイム側の `environment:` か Vault 動的シークレットで注入する。

---

3パターンを選ぶ判断基準をまとめる。個人プロジェクト・ステージング環境ならパターン1で十分だが、本番の外部公開サービスにはパターン3が必須ライン。`VITE_` をつけるのは「ソースコードに直書きしても問題ない値だけ」というルールを CI lint で強制すると、ヒューマンエラーを根本から排除できる。次章ではこの lint を100行のスクリプトに落とし込む。
