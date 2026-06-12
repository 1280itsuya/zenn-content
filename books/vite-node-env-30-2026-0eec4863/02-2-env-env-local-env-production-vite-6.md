---
title: "第2章 .env / .env.local / .env.production｜Viteの読込優先順位と6ファイル早見表"
free: false
---

# 第2章 .env / .env.local / .env.production｜Viteの読込優先順位と6ファイル早見表

## Viteが読む6ファイルの優先順位（高い順に上書きされる）

結論を先に書く。同じ変数が複数ファイルに存在する場合、**`.env.[mode].local` が最優先で勝ち、`.env` が最弱**。`mode` は `--mode` 未指定なら `serve`=development / `build`=production に自動決定される。

| 順位 | ファイル | 読込条件 | git管理 |
|------|---------|----------|---------|
| 1（最強） | `.env.[mode].local` | その mode のみ | 除外 |
| 2 | `.env.[mode]` | その mode のみ | コミット |
| 3 | `.env.local` | 全 mode | 除外 |
| 4（最弱） | `.env` | 全 mode | コミット |

```bash
# 実際の読込順をログ確認（Vite 5.x / 6.x で再現）
npx vite --mode production --debug 2>&1 | grep "env files"
# → [.env, .env.production, .env.local, .env.production.local] の順でマージ
```

## なぜ `.env.local` だけ git 管理外なのか（.gitignore の既定値）

`npm create vite@latest` が生成する `.gitignore` には `*.local` が初期登録されている。これが「`.env.production` はコミットされるのに `.env.production.local` は消える」挙動の正体。

```gitignore
# create-vite が自動生成する行
*.local
```

```bash
# 本番APIキーを誤コミットしていないか即チェック
git ls-files | grep -E '\.env' || echo "OK: env未追跡"
# .env.production がヒットしたら本番キー流出。即 git rm --cached
```

## 「ローカルでは動くのにbuildで消える」=mode取り違えの再現と修正diff

`さとふる` の本番キーを `.env.production` に置いたのに `npm run dev` で `undefined`。原因は dev が `development` mode で `.env.production` を**読まない**こと。

```bash
# 再現：dev では production ファイルが無視される
echo "VITE_SATOFURU_KEY=prod_xxx" > .env.production
npm run dev   # → import.meta.env.VITE_SATOFURU_KEY は undefined
```

```diff
# 修正：環境ごとにファイルを分け、共通の空欄を .env に置く
  .env                  # VITE_SATOFURU_KEY=（空。型補完用）
+ .env.development.local # VITE_SATOFURU_KEY=dev_test_key
+ .env.production        # VITE_SATOFURU_KEY=本番キー（は置かない→下のCI注入へ）
```

## `--mode staging` で第3環境を足す（loadEnv で vite.config.ts から読む）

development/production の2択を超え、検証用 `staging` を追加する。`loadEnv()` を使えば設定ファイル内でも値を参照できる。

```ts
// vite.config.ts
import { defineConfig, loadEnv } from 'vite'

export default defineConfig(({ mode }) => {
  // 第3引数 '' で VITE_ 接頭辞なしも含めて全件ロード
  const env = loadEnv(mode, process.cwd(), '')
  return {
    define: { __API_BASE__: JSON.stringify(env.API_BASE) },
  }
})
```

```bash
# .env.staging を作って起動
echo "API_BASE=https://stg.satofuru.example" > .env.staging
npx vite build --mode staging   # → .env.staging が読まれる
```

## CI/Vercelで本番キーを「コミットせず注入」する4環境統一構成

`.env.production` に本番キーを書かない。CI と Vercel は環境変数を実行時に注入し、ローカル/Docker は `*.local` で吸収する。これで4環境すべてが同一の `import.meta.env.VITE_SATOFURU_KEY` で通る。

```yaml
# .github/workflows/build.yml（GitHub Actions Secrets から注入）
- run: npm run build
  env:
    VITE_SATOFURU_KEY: ${{ secrets.SATOFURU_KEY }}
```

```bash
# Vercel は CLI で本番スコープに登録（ファイル不要）
vercel env add VITE_SATOFURU_KEY production
# Docker はビルド引数で .env.production.local を生成
docker build --build-arg KEY=$KEY -t app .
```

この構成なら本番キーがリポジトリに残らず、`git ls-files | grep .env` は常にコミット可能ファイルだけを返す。次章では `import.meta.env` の型を `vite-env.d.ts` で固め、`undefined` 起因の12エラーのうち4件を TypeScript で事前に潰す。
