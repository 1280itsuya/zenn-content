---
title: "第4章: GitHub Actions上でvite buildが落ちる4パターン — OOM/キャッシュ競合/secrets未到達を実YAMLで解決"
free: false
---

## パターン①: `cache-dependency-path` ハッシュキーミスで旧 Vite 4.x が残留する

**ロスト時間: 平均 47 分 / 発覚難度: 高（ビルドは成功するが出力が壊れる）**

`actions/cache` のキーに `package-lock.json` のパスを誤指定すると、`npm install` をスキップしたまま古いバージョンの Vite がキャッシュから復元される。`vite build` は通るが、バンドルに Vite 4 の出力形式が混入して本番クラッシュする。

```yaml
# ❌ ハッシュが lock ファイルを見ていない
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node

# ✅ lock ファイルの内容変化でキャッシュを必ず無効化
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

確認コマンド:

```bash
# ローカルで Vite バージョンを特定してキャッシュ汚染を検証
npx vite --version
# CI ログの "Cache restored" 行とバージョンが一致するかを突合する
```

---

## パターン②: pnpm workspace と `npm ci` の競合で `vite` コマンドが not found になる

**ロスト時間: 平均 28 分 / エラーログ: `sh: 1: vite: not found`**

pnpm workspace を使うリポジトリに `npm ci` を混在させると、`node_modules/.bin/vite` が生成されない。

```
Run npm run build
> my-app@1.0.0 build
> vite build
sh: 1: vite: not found
Error: Process completed with exit code 127.
```

```yaml
# ❌ pnpm ワークスペースに npm ci を使う
- run: npm ci
- run: npm run build

# ✅ pnpm に統一し workspace も解決
- uses: pnpm/action-setup@v4
  with:
    version: 9
- run: pnpm install --frozen-lockfile
- run: pnpm run build
```

---

## パターン③: 大規模バンドルが OOM SIGKILL で死ぬ — `--max-old-space-size=4096` で突破

**ロスト時間: 平均 63 分（原因特定が最も難しいパターン）**

GitHub Actions の ubuntu-latest は RAM 7 GB だが、Vite の Rollup ワーカーはデフォルトで Node.js ヒープ上限 1.5 GB のまま動く。300 万行超のプロジェクトや `@monaco-editor` 等の巨大依存を含むと SIGKILL が飛ぶ。

```
fatal error: Reached heap limit Allocation failed - JavaScript heap out of memory
 1: 0xb7c6e0 node::Abort() [node]
...
Killed
Error: Process completed with exit code 137.  ← SIGKILL = OOM
```

```yaml
# ✅ NODE_OPTIONS でヒープを拡張 + Rollup の並列数を制限
- name: Build with extended heap
  env:
    NODE_OPTIONS: "--max-old-space-size=4096"
  run: npx vite build --config vite.config.ts

# vite.config.ts 側でも Rollup ワーカー数を絞る
```

```typescript
// vite.config.ts
import { defineConfig } from 'vite'

export default defineConfig({
  build: {
    rollupOptions: {
      maxParallelFileOps: 3,  // デフォルト 20 → 3 に削減
    },
  },
})
```

---

## パターン④: GitHub Secrets が `.env` に降りてこず `import.meta.env.VITE_*` が `undefined`

**ロスト時間: 平均 35 分 / 本番で初めて気づくケースが多い**

`secrets.*` は環境変数として `process.env` に存在するが、Vite は `VITE_` プレフィックスの変数しかクライアントバンドルに埋め込まない。さらに `env:` ブロックに書かないと Vite プロセス自体に変数が届かない。

```yaml
# ❌ secrets をセットせずにビルドする
- run: npx vite build

# ✅ VITE_ プレフィックスで明示的に渡す
- name: Build
  env:
    VITE_API_BASE_URL: ${{ secrets.API_BASE_URL }}
    VITE_SENTRY_DSN: ${{ secrets.SENTRY_DSN }}
  run: npx vite build
```

`undefined` になっていないか CI ログで即確認する方法:

```yaml
- name: Verify env injection
  env:
    VITE_API_BASE_URL: ${{ secrets.API_BASE_URL }}
  run: |
    if [ -z "$VITE_API_BASE_URL" ]; then
      echo "::error::VITE_API_BASE_URL is empty. Check secrets configuration."
      exit 1
    fi
    echo "VITE_API_BASE_URL is set (length: ${#VITE_API_BASE_URL})"
```

---

## 再利用可能な `vite-build.yml` — 4 パターン全対応テンプレート

上記 4 パターンの修正をすべて組み込んだ完成形 YAML。コピーして `.github/workflows/vite-build.yml` に置くだけで動く。

```yaml
name: Vite Build (Production)

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - uses: actions/checkout@v4

      # パターン①対応: lock ファイルをハッシュキーに含める
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - uses: actions/cache@v4
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-

      - run: npm ci

      # パターン④対応: secrets を VITE_ プレフィックスで注入
      - name: Verify secrets
        env:
          VITE_API_BASE_URL: ${{ secrets.API_BASE_URL }}
        run: |
          if [ -z "$VITE_API_BASE_URL" ]; then
            echo "::error::VITE_API_BASE_URL is empty"
            exit 1
          fi

      # パターン③対応: OOM 対策でヒープ拡張
      - name: Build
        env:
          NODE_OPTIONS: "--max-old-space-size=4096"
          VITE_API_BASE_URL: ${{ secrets.API_BASE_URL }}
          VITE_SENTRY_DSN: ${{ secrets.SENTRY_DSN }}
        run: npm run build

      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/
          retention-days: 7
```

パターン②(pnpm ワークスペース) は `npm ci` 行を以下と差し替える:

```yaml
      - uses: pnpm/action-setup@v4
        with:
          version: 9
      - run: pnpm install --frozen-lockfile
```

この YAML 1 枚で本章の 4 パターン全員を封じられる。CI ログで `exit code 137` や `vite: not found` を見たときは、このテンプレートと自分の YAML を diff して原因を特定する。
