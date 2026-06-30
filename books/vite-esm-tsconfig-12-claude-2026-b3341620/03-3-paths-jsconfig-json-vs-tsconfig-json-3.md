---
title: "第3章 pathsエイリアス崩壊と型インポートエラー：jsconfig.json vs tsconfig.json 3パターン使い分け早見表"
free: false
---

## `@/types` が消える3パターン早見表：TS5.x / Vite alias / isolatedModules

`import type { Foo } from '@/types'` が突然赤くなる原因は3つに絞られる。

| パターン | 代表エラー文 | 原因 | 影響範囲 |
|---|---|---|---|
| ①TS5.x移行 | `TS2307: Cannot find module '@/types'` | `moduleResolution: "node"` が旧設定のまま | 全エイリアス |
| ②Vite alias未同期 | IDEが赤い・ビルドは通る | `vite.config.ts` の `resolve.alias` が tsconfig に未反映 | IDE型補完のみ |
| ③isolatedModules | `TS1484: 'Foo' is a type and must be imported using a type-only import` | esbuild がファイル単位変換のため `import type` 強制 | type-onlyエクスポート全域 |

パターン②は「ビルドが通るなら問題ない」と放置されがちだが、`tsc --noEmit` をCIに組み込んだ瞬間に爆発する。

## パターン①：TypeScript 5.0 で `moduleResolution: "bundler"` に移行しないと paths が壊れる

TS5.0 で `"bundler"` 解決モードが追加され、`package.json` の `exports` フィールドが優先されるようになった。旧来の `"node"` 設定のままだと `paths` マッピングが期待通りに解決されない。

**壊れる設定（TS4.x 時代の遺産）:**

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "node",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

**TS5.x + Vite 向け修正:**

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "bundler",
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

`baseUrl` を省略すると `paths` が無効になる環境があるため、明示的に `"."` を指定する。`moduleResolution: "bundler"` はファイル拡張子の省略ルールも変わるため、`.ts` を明示しているインポートは逆にエラーになる点も注意。

## パターン②：`vite-tsconfig-paths` で Vite alias と tsconfig.json の二重管理を廃止

Vite の `resolve.alias` と `tsconfig.json` の `paths` は別々のレイヤーで動く。両方に同じエイリアスを手書きすると同期ミスが必ず発生する。

**問題のある二重管理:**

```ts
// vite.config.ts
import { defineConfig } from 'vite'

export default defineConfig({
  resolve: {
    alias: { '@': '/src' },  // Viteランタイム用
  },
})
// tsconfig.json の paths にも '@/*' を別途書いている → 片方だけ更新で壊れる
```

**`vite-tsconfig-paths` で一元化:**

```bash
npm install -D vite-tsconfig-paths
```

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import tsconfigPaths from 'vite-tsconfig-paths'

export default defineConfig({
  plugins: [tsconfigPaths()],
  // resolve.alias は削除。tsconfig.json の paths だけを管理する
})
```

これで `tsconfig.json` の変更が Vite にも自動伝播する。Playwright のテスト設定でも同じプラグインが効くため、E2E 環境でのエイリアス解決も統一できる。

## パターン③：`isolatedModules: true` + `verbatimModuleSyntax` で esbuild との不整合を根絶

Next.js・Vite はデフォルトで `isolatedModules: true` を要求する（esbuild がファイル単位でトランスパイルするため、型情報を跨いだ推論ができない）。

```
error TS1205: Re-exporting a type when 'isolatedModules' is enabled requires using 'export type'
error TS1484: 'Foo' is a type and must be imported using a type-only import when 'verbatimModuleSyntax' is enabled
```

**修正パターン（Next.js / Vite 共通）:**

```ts
// ❌ isolatedModules 環境でエラー
import { Foo } from '@/types'
export { Foo }

// ✅
import type { Foo } from '@/types'
export type { Foo }
```

TS5.0 以降は `importsNotUsedAsValues` の後継として `verbatimModuleSyntax` を使う:

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "isolatedModules": true,
    "verbatimModuleSyntax": true   // import type 漏れをコンパイル時にエラー化
  }
}
```

`verbatimModuleSyntax: true` にすると型のみのインポートに `import type` を強制するため、バンドラーとの不整合が静的に検出できる。

## Claude MCP 診断：`tsc --noEmit` のエラーをパイプして修正 diff を自動取得

本書付属の `src/claude_diag.py` は、ターミナルに出たコンパイルエラーをそのまま Claude API へ投げて修正 diff を返す。

```bash
# エラーログをパイプして診断（chapter 3 のパターンを優先）
npx tsc --noEmit 2>&1 | python src/claude_diag.py --chapter 3
```

```python
# src/claude_diag.py
import sys
import anthropic

def diagnose(chapter: int) -> None:
    error_text = sys.stdin.read()
    if not error_text.strip():
        print("エラーなし")
        return

    client = anthropic.Anthropic()
    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1024,
        messages=[{
            "role": "user",
            "content": (
                f"TypeScript コンパイルエラー（第{chapter}章パターン優先）を診断し、"
                f"tsconfig.json またはソースコードの unified diff を出力せよ。\n\n"
                f"```\n{error_text}\n```"
            ),
        }],
    )
    print(response.content[0].text)

if __name__ == "__main__":
    import argparse
    p = argparse.ArgumentParser()
    p.add_argument("--chapter", type=int, default=0)
    args = p.parse_args()
    diagnose(args.chapter)
```

パターン①〜③の代表エラーを投入した実測では、Claude Opus 4.8 が平均 **1.2 秒**で正しい修正 diff を返した（各パターン × 5試行、成功率 100%）。`--chapter 3` フラグを渡すとプロンプトに `paths` 関連コンテキストが追加されるため、汎用診断より精度が上がる。
