---
title: "第5章 GitHub Actions Secrets→Vercel本番｜CI/CDでenvが空になる5原因の逆引き"
free: false
---

第5章 GitHub Actions Secrets→Vercel本番｜CI/CDでenvが空になる5原因の逆引き

---

CI/CDで`import.meta.env.VITE_API_URL`が`undefined`になる障害は、ローカルとDockerが通った後の最後の壁になる。原因はほぼ5パターンに収束し、エラーログの文言で逆引きできる。本章はGitHub Actions Secrets→Vercel本番までの実ログ5種を、そのまま貼れる`workflow.yml`とチートシートで潰す。

## 原因1: Secrets未登録｜`Error: VITE_API_URL is not set`の逆引き

ビルドログに`is not set`や空文字が出るなら、まず`secrets`がActionsに渡っていない。`env:`へ明示マッピングするのが確実で、`${{ secrets.X }}`をstepの`env`に展開する。

```yaml
# .github/workflows/deploy.yml
jobs:
  build:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run build
        env:
          VITE_API_URL: ${{ secrets.VITE_API_URL }}   # stepのenvへ必ず展開
```

登録確認はCLIで一覧でき、名前のtypoを即発見できる。

```bash
gh secret list --repo your/repo
# VITE_API_URL  Updated 2026-06-10
```

## 原因2: 環境スコープ違い｜Environment secretsがjobに来ない

`gh secret list`に出るのにビルドで空なら、Repository secretsではなく **Environment secrets** に登録されている。jobに`environment:`を宣言しないと、その環境のsecretは注入されない。

```yaml
jobs:
  build:
    runs-on: ubuntu-22.04
    environment: production   # この1行が無いとproduction環境のsecretは空
    steps:
      - run: npm run build
        env:
          VITE_API_URL: ${{ secrets.VITE_API_URL }}
```

## 原因3: VITE_接頭辞漏れ｜`API_URL`はクライアントに出ない

Viteはセキュリティ上`VITE_`接頭辞の変数だけをクライアントバンドルに露出する。`API_URL`のままだと`import.meta.env.API_URL`は永久に`undefined`になる。`vite.config.ts`の`envPrefix`で挙動を確認する。

```ts
// vite.config.ts
import { defineConfig } from 'vite'
export default defineConfig({
  envPrefix: 'VITE_', // 既定値。これ以外の名前はビルドに焼かれない
})
```

```ts
// NG: undefined  →  OK: 値が入る
console.log(import.meta.env.API_URL)       // undefined
console.log(import.meta.env.VITE_API_URL)  // "https://api.example.com"
```

## 原因4: pull_request実行のsecret制限｜forkで値が空になる

forkからの`pull_request`イベントはsecretsが意図的にブロックされ、空文字になる。ログに変数名だけ出て値が空なら、トリガーを`push`か`pull_request_target`へ切り分ける。

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
jobs:
  build:
    # forkのPRではsecretを使うstepをスキップし、ビルド失敗を回避
    if: github.event_name == 'push' || github.event.pull_request.head.repo.full_name == github.repository
```

## 原因5: Vercel本番のEnvironment未設定｜Previewは通るが本番だけ落ちる

デプロイは成功するのに本番だけ`undefined`──これはVercelのEnvironmentスコープ漏れだ。`Production`にチェックを入れ忘れ`Preview`のみ登録した実例で、`vercel env ls`を見れば一目で分かる。

```bash
vercel env ls
# VITE_API_URL   Encrypted   Preview          ← Productionが無い=本番で空
vercel env add VITE_API_URL production         # 本番スコープへ追加
vercel pull --environment=production .env.production.local  # ローカルへ同期
```

## 4環境チートシート｜ローカル/Docker/CI/Vercelを1枚で管理

最後に、同じ`VITE_API_URL`を4環境で齟齬なく通すコピペ用の対応表を置く。どこに書けば値が届くかが環境ごとに違う点が全障害の根だ。

```bash
# ① ローカル        : .env.local                （gitignore対象）
# ② Docker build    : Dockerfile ARG/ENV + --build-arg
# ③ GitHub Actions  : Settings>Secrets + workflow env: マッピング
# ④ Vercel 本番     : Environment=Production にチェック + vercel pull

# 検証ワンライナー（4環境共通で値がビルドに焼かれたか確認）
grep -r "VITE_API_URL" dist/assets/*.js | head -1
# → 値がhitすればビルド反映済み。空ならスコープ漏れを上の5原因で逆引き
```

最終行の`grep`を4環境すべてで回せば、`dist`に値が焼けたかを2秒で判定できる。空が返った環境だけを原因1〜5へ機械的に当てはめれば、CI/CDのenv空問題は逆引きで確定する。
