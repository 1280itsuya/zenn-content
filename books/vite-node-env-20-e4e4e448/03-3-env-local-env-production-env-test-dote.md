---
title: "第3章 .env.local / .env.production / .env.test — 優先順位マトリクスと dotenv-expand 変数展開バグの修正パターン"
free: false
---

Zenn有料章を執筆します。

## 第3章 .env.local / .env.production / .env.test — 優先順位マトリクスと dotenv-expand 変数展開バグの修正パターン

---

## Vite 公式ソース (`packages/vite/src/node/env.ts`) で確認する5ファイルのロード順

Vite は `loadEnv()` 内で以下の順に `fs.readFileSync` を試みる。**後ろのファイルが前を上書きする**。

| 優先度 | ファイル名 | `.env.test` 実行時 | 備考 |
|--------|------------|-------------------|------|
| 1（最低）| `.env` | ロード | 全モード共通 |
| 2 | `.env.local` | **スキップ** | `test` モードは意図的に除外 |
| 3 | `.env.[mode]` | `.env.test` | モード専用 |
| 4（最高）| `.env.[mode].local` | `.env.test.local` | gitignore 推奨 |

`test` モードで `.env.local` がスキップされる理由は、Vitest 実行中に開発者のローカル秘密鍵が混入するのを防ぐため。Vite v5.1 以降はこの挙動が明示的にドキュメント化されている。

```bash
# 再現リポジトリ: git clone 後すぐ動作確認
cat .env          # VITE_API=base
cat .env.local    # VITE_API=local
cat .env.test     # VITE_API=test

# 期待値確認
npx vite build --mode production  # => base or local が勝つ
npx vitest run                    # => test が勝つ (.env.local は無視)
```

---

## dotenv-expand の展開順序バグ: `$BASE_URL` が空文字になるパターン

dotenv-expand v10 以降、**同一ファイル内で定義前の変数を参照すると展開されない**既知の問題がある。

```dotenv
# NG: .env — BASE_URLより先にAPIが定義されている
VITE_API_URL=$BASE_URL/api   # ← この時点でBASE_URLは未定義 → 展開されず空
BASE_URL=https://example.com
```

```dotenv
# OK: 参照より先に定義する
BASE_URL=https://example.com
VITE_API_URL=$BASE_URL/api   # => https://example.com/api
```

`vite.config.ts` 側でフォールバックを噛ませると CI での無言の空文字バグを防げる。

```typescript
// vite.config.ts
import { defineConfig, loadEnv } from 'vite'

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '')
  if (!env.VITE_API_URL) {
    throw new Error(`VITE_API_URL is empty. Check .env expansion order. mode=${mode}`)
  }
  return { /* ... */ }
})
```

---

## Turborepo / Nx モノレポで `.env` が読まれない3パターンと修正 diff

モノレポでは `loadEnv()` の第2引数 `envDir` が **`packages/web/` ではなくリポジトリルート** を指すケースで詰まる。

```
repo-root/
├── .env              ← ここにある
├── packages/
│   └── web/
│       ├── vite.config.ts  ← envDir のデフォルトは process.cwd() = packages/web/
│       └── .env            ← ここを見ている (ファイルがないので無視)
```

**修正 diff:**

```diff
// packages/web/vite.config.ts
- const env = loadEnv(mode, process.cwd(), '')
+ const env = loadEnv(mode, path.resolve(__dirname, '../..'), '')
```

Turborepo の `turbo.json` でパスを明示する場合:

```json
{
  "pipeline": {
    "build": {
      "env": ["VITE_API_URL", "VITE_SENTRY_DSN"],
      "outputs": ["dist/**"]
    }
  }
}
```

`env` フィールドに列挙しないと Turborepo のキャッシュが env 変化を検知せず、**古いビルド結果を返し続ける**。

---

## CI での env ファイル責任分離パターン3種比較

| パターン | 生成タイミング | 管理主体 | 向く規模 |
|----------|--------------|---------|---------|
| A: Secrets → `echo` で `.env` 生成 | ジョブ実行時 | CI 管理者 | 小〜中 |
| B: Vault/1Password CLI でプル | ジョブ実行時 | SecOps | 中〜大 |
| C: `.env.ci` をリポジトリにコミット | コミット時 | 開発者全員 | ライブラリ/OSS |

**パターン A の実装（GitHub Actions）:**

```yaml
# .github/workflows/build.yml
jobs:
  build:
    steps:
      - uses: actions/checkout@v4
      - name: Generate .env.production
        run: |
          cat <<EOF > .env.production
          VITE_API_URL=${{ secrets.API_URL }}
          VITE_SENTRY_DSN=${{ secrets.SENTRY_DSN }}
          EOF
      - run: npm run build
```

パターン C でコミットする `.env.ci` には `VITE_` プレフィックス変数のみ許可し、`SECRET_` を含む行が混入したら CI を即座に落とす検証ステップを挟む。

```bash
# CI 自動検出スクリプト (100行の一部)
# check-env-secrets.sh
grep -E '^(SECRET_|AWS_|DATABASE_URL)' .env.ci && {
  echo "::error::Sensitive key detected in .env.ci — use GitHub Secrets instead"
  exit 1
}
```

---

## チーム運用で衝突しないディレクトリ設計: `.gitignore` 3行ルール

```
# .gitignore に追加する3行
.env.local
.env.*.local
.env.production   # 本番値は絶対にコミットしない
```

コミットして良いファイルは `.env`（ダミー値）・`.env.development`（ローカル開発用デフォルト）・`.env.ci`（CI 専用非秘密変数）の3種に限定する。`VITE_` プレフィックスがないキーはブラウザに公開されないが、**ビルドログに平文で出力されるリスク**があるため、CI ログマスク設定と組み合わせること。
