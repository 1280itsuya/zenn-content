---
title: "第4章 Docker Composeとマルチステージbuild｜ビルド時に消えるVITE_変数の通し方"
free: false
---

## `VITE_API_URL is undefined` がruntimeで消える境界線

結論：Viteの`import.meta.env.VITE_*`は`vite build`時に文字列リテラルへ静的展開されるため、Docker runの`-e`やcomposeの`environment:`で後から注入しても本番バンドルには反映されない。出るエラーは決まってこの3つだ。

```bash
# よく出る3メッセージと発生タイミング
# 1) Uncaught TypeError: Cannot read properties of undefined (reading 'API_URL')  → build時に値が空
# 2) net::ERR_NAME_NOT_RESOLVED  → "undefined/api" へfetchして名前解決失敗
# 3) [vite] define is not allowed to be used  → defineで動的注入を試みた残骸
docker run -e VITE_API_URL=https://api.example.com app   # ← これは効かない(build後だから)
```

## build-arg経由でVITE_を焼き込むDockerfile(マルチステージ)

`ARG`→`ENV`の橋渡しが命綱…ではなく必須条件。`ARG`はビルドレイヤー限定なので、`vite build`が読む`ENV`へ昇格させてから実行する。

```dockerfile
# syntax=docker/dockerfile:1.7
FROM node:22.11-slim AS build
WORKDIR /app
ARG VITE_API_URL                      # compose/CIから受け取る
ENV VITE_API_URL=${VITE_API_URL}      # vite buildが参照できる形に昇格
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build                      # ここでVITE_API_URLがdist内に静的展開される

FROM nginx:1.27-alpine AS runtime
COPY --from=build /app/dist /usr/share/nginx/html
# runtimeステージにソースも.envも残らない=秘密が本番イメージに混入しない
```

## docker-compose.ymlの`args`と`env_file`の決定的な差

`environment:`はコンテナ実行時、`build.args:`はビルド時。VITE_変数は**100%** `build.args`側に置く。`env_file`をbuildに効かせるには`args`へ明示展開が要る。

```yaml
# docker-compose.yml
services:
  web:
    build:
      context: .
      args:
        VITE_API_URL: ${VITE_API_URL}   # ホスト or CIの環境変数を注入
    env_file:
      - .env                            # ← これはbuild時には使われない(実行時のみ)
    ports:
      - "8080:80"
# 起動: VITE_API_URL=https://api.example.com docker compose up --build
```

## dotenvxで`.env`を暗号化同梱し復号鍵だけ注入する

`.env`を平文でCOPYするとイメージ解析で漏れる。dotenvx(`@dotenvx/dotenvx`)なら暗号化済み`.env`をリポジトリ同梱し、復号鍵`DOTENV_PRIVATE_KEY`だけをbuild-argで渡す。

```dockerfile
FROM node:22.11-slim AS build
WORKDIR /app
ARG DOTENV_PRIVATE_KEY                 # 鍵だけ注入(.env.keysはイメージに入れない)
ENV DOTENV_PRIVATE_KEY=${DOTENV_PRIVATE_KEY}
COPY . .                               # 暗号化済み.envはコミット済みでOK
RUN npm ci
RUN npx @dotenvx/dotenvx run -- npm run build   # 復号→VITE_展開→破棄
```

```bash
# 暗号化(ローカル一度だけ): .env が暗号文に置換され .env.keys に秘密鍵が出る
npx @dotenvx/dotenvx encrypt
docker build --build-arg DOTENV_PRIVATE_KEY=$(grep -oP '(?<==).*' .env.keys) -t app .
```

## ローカル/Docker/CI/Vercel 4環境で同じ`.env`を通す

同一の`.env`を4環境で破綻なく回す対応表。VITE_変数は常にbuild時、鍵は環境のSecret機構で渡すのが共通解。

```yaml
# .github/workflows/build.yml (CI)
- run: |
    docker build \
      --build-arg VITE_API_URL=${{ vars.VITE_API_URL }} \
      --build-arg DOTENV_PRIVATE_KEY=${{ secrets.DOTENV_PRIVATE_KEY }} \
      -t app .
```

| 環境 | VITE_変数の渡し方 | 復号鍵の置き場 |
|------|------------------|----------------|
| ローカル | `.env`をdotenvxが復号 | `.env.keys`(gitignore) |
| Docker | `--build-arg` | `--build-arg` |
| CI | `vars.*` | `secrets.DOTENV_PRIVATE_KEY` |
| Vercel | Project→Environment Variables | 同左(Encrypted) |

Vercelはbuild環境にVITE_変数を自動注入するため`build.args`は不要だが、Docker/CIと値をずらすと`ERR_NAME_NOT_RESOLVED`が再発する。4環境で同じキー名・同じ値を一元管理することが事故ゼロの条件だ。
