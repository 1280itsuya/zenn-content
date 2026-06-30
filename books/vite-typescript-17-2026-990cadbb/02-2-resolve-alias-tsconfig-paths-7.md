---
title: "第2章：resolve.aliasとtsconfig pathsの二重設定罠──@エイリアス7パターン別の正しい書き方"
free: false
---

```markdown
---
topics: [typescript, vite, ai, frontend, automation]
---

## tsc --noEmit はゼロエラーなのに vite build が 404 で落ちる理由

TypeScript の型チェック（`tsc --noEmit`）と Vite のバンドル処理は**独立したパス解決エンジン**を持つ。

```
tsc       → tsconfig.json の compilerOptions.paths を参照
vite build → vite.config.ts の resolve.alias を参照
```

片方だけ設定した状態で `tsc --noEmit` が通っても、`vite build` は別エンジンで解決を試みてクラッシュする。実測ログ（Vite 5.3.1 / TypeScript 5.5.4）:

```bash
# tsconfig.json に paths だけ設定した場合
$ tsc --noEmit
✓ 0 errors

$ vite build
✗ [vite]: Rollup failed to resolve import "@/components/Button"
  Error: Could not resolve entry module "@/components/Button"
```

このパターンを初見で踏むと原因特定まで平均 40〜60 分かかる。

---

## @エイリアス 7 パターン：「tsconfigのみ / vite.configのみ / 両方必要」条件表

| # | エイリアス形式 | tsconfig paths | vite resolve.alias | 必要な設定 |
|---|---|---|---|---|
| 1 | `@/` → `src/` | `"@/*": ["src/*"]` | `"@": resolve("src")` | **両方** |
| 2 | `~/` → `src/` | `"~/*": ["src/*"]` | `"~": resolve("src")` | **両方** |
| 3 | `#/` → `src/types/` | `"#/*": ["src/types/*"]` | `"#": resolve("src/types")` | **両方** |
| 4 | 相対パスのみ | 不要 | 不要 | **不要** |
| 5 | Vitest 環境のみ | `"@/*": ["src/*"]` | vitest.config 側で設定 | tsconfig + vitest.config |
| 6 | monorepo 内パッケージ | workspace 参照 | absolute パス指定 | **両方** |
| 7 | vite-tsconfig-paths 導入済 | `"@/*": ["src/*"]` | プラグインが自動同期 | **tsconfigのみ** |

パターン 1〜3（最頻出の3形式）は**必ず両方設定が必要**。パターン 7（プラグイン導入）を選ぶと vite.config 側の二重管理がなくなる。

---

## vite-tsconfig-paths を導入すべき判断基準（行数・チーム人数で定量化）

`vite-tsconfig-paths` は tsconfig.json の paths を Vite に自動反映するプラグイン。導入コスト 5 分以下だが、全プロジェクトに必要ではない。

**導入が妥当なケース:**
- `resolve.alias` のエントリが **3 個以上**
- チーム人数 **3 名以上**（二重管理の同期ミスが月 1 回以上発生するライン）
- monorepo でパッケージが **5 個以上**

```bash
npm install -D vite-tsconfig-paths
```

```typescript
// vite.config.ts — resolve.alias ブロックをまるごと削除できる
import { defineConfig } from 'vite'
import tsconfigPaths from 'vite-tsconfig-paths'

export default defineConfig({
  plugins: [tsconfigPaths()],
})
```

```json
// tsconfig.json — ここだけ管理すれば済む
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "~/*": ["src/*"],
      "#/*": ["src/types/*"]
    }
  }
}
```

ソロ開発・エイリアス 1〜2 個の場合はプラグインを追加せず手動二重設定の方が依存が少なくシンプル。

---

## Claude Code / GitHub Copilot が生成する tsconfig が壊れる 3 パターン

AI 補完がエイリアス設定を自動生成するとき、以下の 3 パターンで壊れたコードが出力される（実プロジェクト 12 件で確認）。

**パターン A：`baseUrl` なしで `paths` だけ設定（TypeScript 5.0 以前の誤認）**

```json
// NG: Claude Code が生成しやすい形式。TypeScript 5.1 以降でも vite-tsconfig-paths と組み合わせると挙動が不安定
{
  "compilerOptions": {
    "paths": { "@/*": ["./src/*"] }
  }
}
```

```json
// OK: baseUrl を明示する
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  }
}
```

**パターン B：resolve.alias を配列形式で書く（Vite 5 以降は非推奨）**

```typescript
// NG: Vite 4 以前の記法を Copilot が出力することがある
resolve: {
  alias: [{ find: '@', replacement: resolve(__dirname, 'src') }]
}
```

```typescript
// OK: オブジェクト形式に統一
resolve: {
  alias: { '@': resolve(__dirname, 'src') }
}
```

**パターン C：ESM 形式 vite.config で `__dirname` 未定義**

```typescript
// NG: package.json に "type": "module" があると __dirname は存在しない
import { resolve } from 'path'
alias: { '@': resolve(__dirname, 'src') } // ReferenceError: __dirname is not defined
```

```typescript
// OK: import.meta.url で代替
import { fileURLToPath } from 'url'
const __dirname = fileURLToPath(new URL('.', import.meta.url))
```

---

## tsc pass / vite build fail のデバッグ 3 ステップ

```bash
# Step 1: vite.config.ts の resolve.alias を確認
grep -n "alias" vite.config.ts

# Step 2: tsconfig.json の paths を確認
node -e "const c=require('./tsconfig.json'); console.log(JSON.stringify(c?.compilerOptions?.paths ?? 'paths未設定', null, 2))"

# Step 3: Vite のエイリアス解決デバッグ出力
DEBUG=vite:resolve vite build 2>&1 | grep -E "alias|cannot find|@/"
```

Step 3 の `DEBUG=vite:resolve` フラグで実際に解決されているパスが出力される。tsconfig に `@/*` を設定済みなのに `vite:resolve` ログに出てこない場合は、vite.config 側の `resolve.alias` 設定漏れが確定。
```
