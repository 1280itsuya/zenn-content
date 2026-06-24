---
title: "第2章 import.meta.env vs process.env — Vite/Node 二重環境の完全仕様マップと TypeScript 型定義テンプレート"
free: false
---

## ESBuild が import.meta.env を「静的置換」する仕組み — Vite 5.x の変換コードを読む

Vite は内部で ESBuild を使い、`import.meta.env.VITE_*` を**バンドル時に文字列リテラルへ置換**する。`.env` を読み込むのは Node.js プロセス（Vite dev server）であり、ブラウザには値そのものが埋め込まれる。

```bash
# ビルド後の dist/assets/index-xxxxx.js で置換済み文字列を確認する
grep -oE '"VITE_API_URL":"[^"]+"' dist/assets/index-*.js
# 出力例: "VITE_API_URL":"https://api.example.com"
```

`define` オプションでその正体を手動再現できる:

```ts
// vite.config.ts — Vite が内部でやっている静的置換を明示する
import { defineConfig } from 'vite'

export default defineConfig({
  define: {
    'import.meta.env.VITE_API_URL': JSON.stringify(process.env.VITE_API_URL ?? ''),
  },
})
```

ビルド後のバンドルに `import.meta.env` というキーワードは一切残らない。

---

## process.env との決定的差異 — ランタイム解決 vs バンドル時静的展開

| 比較軸 | `import.meta.env.VITE_*` | `process.env.*` |
|---|---|---|
| 解決タイミング | **バンドル時**（ESBuild define） | **ランタイム**（Node.js グローバル） |
| ブラウザ到達 | ○ 値がバンドルに埋め込まれる | ✕ `process` オブジェクト自体が存在しない |
| 秘密情報の扱い | `VITE_` 接頭辞なしは自動除外 | 全変数が参照可能（漏洩リスクあり） |
| 動的更新 | 不可（再ビルド必須） | 可（環境変数を変えれば即時） |

```ts
// ❌ サーバーサイドのシークレットをフロントで誤参照するパターン
const secret = import.meta.env.DATABASE_URL  // → undefined（VITE_ 接頭辞なし）

// ✅ フロントから参照できるのは VITE_ 接頭辞付きのみ
const apiUrl = import.meta.env.VITE_API_URL  // → "https://api.example.com"

// ✅ Node.js（Express / Fastify）では process.env を使う
const dbUrl = process.env.DATABASE_URL       // → ランタイムで解決
```

---

## フロント/サーバー共通コードの env 参照フローチャート

```
コードはブラウザで実行されるか？
├─ YES → VITE_ 接頭辞付き変数のみ使用
│         import.meta.env.VITE_FOO
│         └─ .env に VITE_FOO=xxx を記述
└─ NO（Node.js サーバーサイド）
          ├─ Vite SSR モード（vite build --ssr）？
          │   └─ YES → import.meta.env はサーバーでも参照可
          │             ただし VITE_ なし変数も expose される点に注意
          │             secrets は process.env に統一するのが安全
          └─ NO（Express / Next.js API Routes / Cloud Functions）
                    → process.env.SECRET_KEY を使用
                      dotenv.config() を最上部で一度だけ呼ぶ
```

```ts
// SSR で両方を安全に扱う分岐パターン
const isServer = typeof window === 'undefined'

export function getApiUrl(): string {
  if (isServer) {
    return process.env.API_URL ?? ''          // Node.js ランタイム
  }
  return import.meta.env.VITE_API_URL ?? ''  // ブラウザバンドル
}
```

---

## TypeScript 型定義テンプレート — ImportMetaEnv 拡張で型エラーを根絶する

`src/vite-env.d.ts` を拡張する。プロジェクト全体で共有できるコピペテンプレート:

```ts
// src/vite-env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  // ─── 公開変数（VITE_ 接頭辞必須） ───────────────────────
  readonly VITE_API_URL: string
  readonly VITE_APP_NAME: string
  readonly VITE_FEATURE_FLAG_DARK_MODE: 'true' | 'false'
  // ─── 追加する変数はここへ ────────────────────────────────
}

interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

```jsonc
// tsconfig.json — vite-env.d.ts を確実に参照させる
{
  "compilerOptions": {
    "types": ["vite/client"]
  },
  "include": ["src/**/*.ts", "src/**/*.tsx", "src/vite-env.d.ts"]
}
```

`ImportMetaEnv` に未登録の変数を参照すると**コンパイルエラー**になり、ランタイムで `undefined` に気づく前に IDE が警告する。`VITE_TYPO` のような typo がビルドを通過しなくなる。

---

## 既存コードの env 参照を一括検出する Node スクリプト 100 行

既存プロジェクトへ貼り付けてすぐ動く。`process.env.VITE_*`（フロントでの誤用）と `import.meta.env` の `VITE_` なし参照（秘密情報漏洩リスク）を同時に検出する:

```js
// scripts/check-env-refs.mjs
import { readFileSync, readdirSync, statSync } from 'fs'
import { join, extname } from 'path'

const TARGET_EXTS = new Set(['.ts', '.tsx', '.js', '.jsx', '.vue', '.svelte'])
const IGNORE_DIRS = new Set(['node_modules', 'dist', '.git', '.vite'])

const PATTERNS = [
  {
    name: 'process.env.VITE_* in frontend (should use import.meta.env)',
    regex: /process\.env\.(VITE_\w+)/g,
    severity: 'error',
  },
  {
    name: 'import.meta.env without VITE_ prefix (secret exposure risk)',
    regex: /import\.meta\.env\.(?!VITE_|MODE|BASE_URL|PROD|DEV|SSR)(\w+)/g,
    severity: 'warn',
  },
  {
    name: 'hardcoded empty string fallback (silent failure)',
    regex: /process\.env\.\w+\s*\|\|\s*['"]{2}/g,
    severity: 'info',
  },
]

function walk(dir, files = []) {
  for (const entry of readdirSync(dir)) {
    if (IGNORE_DIRS.has(entry)) continue
    const full = join(dir, entry)
    if (statSync(full).isDirectory()) walk(full, files)
    else if (TARGET_EXTS.has(extname(entry))) files.push(full)
  }
  return files
}

const root = process.argv[2] ?? '.'
const files = walk(root)
let errorCount = 0, warnCount = 0

for (const file of files) {
  const src = readFileSync(file, 'utf8')
  const lines = src.split('\n')
  for (const { name, regex, severity } of PATTERNS) {
    for (const [lineIdx, line] of lines.entries()) {
      for (const match of line.matchAll(regex)) {
        const label = severity === 'error' ? '❌ ERROR'
          : severity === 'warn' ? '⚠️  WARN' : 'ℹ️  INFO'
        console.log(`${label}  ${file}:${lineIdx + 1}`)
        console.log(`       Rule : ${name}`)
        console.log(`       Match: ${match[0]}\n`)
        if (severity === 'error') errorCount++
        else if (severity === 'warn') warnCount++
      }
    }
  }
}

console.log('─'.repeat(50))
console.log(`Checked ${files.length} files`)
console.log(`❌ Errors: ${errorCount}  ⚠️  Warnings: ${warnCount}`)
if (errorCount > 0) process.exit(1)
```

```bash
# ローカル実行
node scripts/check-env-refs.mjs src/

# GitHub Actions に組み込む例
- name: Check env refs
  run: node scripts/check-env-refs.mjs src/
```

このスクリプトを `package.json` の `lint:env` に追加し、CI の `lint` ジョブに差し込むだけで、チーム全体の env 参照ミスをコードレビュー前に検出できる。第3章では `.env` ファイルが git に混入した場合の**漏洩検出と即時無効化フロー**を扱う。
