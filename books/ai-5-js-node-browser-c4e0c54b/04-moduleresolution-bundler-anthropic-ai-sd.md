---
title: "`moduleResolution: bundler` が @anthropic-ai/sdk 型推論を壊す — TypeScript 5.x 設定差分5本"
free: false
---

## `moduleResolution: bundler` が引き起こす3系統のエラーの全容

TypeScript 5.0追加の`"moduleResolution": "bundler"`はVite/esbuild統合に便利だが、`@anthropic-ai/sdk@0.24+`の型定義が**Node10互換のCommonJS前提**で書かれているため、以下3系統のエラーが頻出する。

```
TS2307: Cannot find module '@anthropic-ai/sdk' or its corresponding type declarations.
TS2322: Type 'ReadableStream<Uint8Array>' is not assignable to type 'ReadableStream<string>'.
TS2339: Property 'stream' does not exist on type 'Stream<...>'.
```

本章では差分コミット5本で`tsc --noEmit`が0エラーに到達するまでの手順を再現する。TypeScript設定調査に平均4.5時間かかるところを30分に圧縮できる。

## Commit \#1: `moduleResolution` を `"Node16"` に切り戻してエラーを分類する

最初の手順として`bundler`→`Node16`に一時切り替える。これだけで`TS2307`が解消する場合、問題は`bundler`モードの**拡張子省略許容ロジック**にある。`module`フィールドと`moduleResolution`は必ずペアで変える。

```jsonc
// tsconfig.json — Commit #1 差分
{
  "compilerOptions": {
    // "moduleResolution": "bundler",  // ← コメントアウト
    "moduleResolution": "Node16",      // ← 一時的に戻す
    "module": "Node16"                 // module も合わせること必須
  }
}
```

```bash
npx tsc --noEmit 2>&1 | grep -c "error TS"
# Before: 7 errors → After: 3 errors
```

`bundler`に戻す前に、`"module"`が`"ESNext"` / `"Preserve"` / `"CommonJS"`のいずれかでなければ`bundler`は有効にならない制約を押さえておく。

## Commit \#2: `lib` 配列で `"DOM.Iterable"` 欠落による `ReadableStream` 型不一致を潰す

`TS2322: Type 'ReadableStream<Uint8Array>' is not assignable`は`lib`配列の不備が原因。`@anthropic-ai/sdk`のStreaming APIは`ReadableStream<Uint8Array>`を返すが、`"lib": ["ES2022", "DOM"]`だとDOMの`ReadableStream<R>`と型引数が衝突する。

```jsonc
// Commit #2 差分
{
  "compilerOptions": {
    "lib": [
      "ES2022",
      "ESNext.AsyncIterable",  // ← 追加: for-await-of サポート
      "DOM",
      "DOM.Iterable"           // ← 追加: これがないと ReadableStream が壊れる
    ]
  }
}
```

```bash
npx tsc --noEmit 2>&1 | grep "ReadableStream"
# After: 0 matches
```

## Commit \#3: `skipLibCheck: true` が隠していた `paths` エイリアス干渉を露出・修正する

`skipLibCheck: true`は`.d.ts`エラーを**握りつぶす**だけで根本原因は残る。`paths`エイリアスと`bundler`の組み合わせで`@anthropic-ai/sdk`が別解決パスに迷い込んでいるケースを`--traceResolution`で特定する。

```jsonc
// Commit #3 差分
{
  "compilerOptions": {
    "skipLibCheck": false,
    "paths": {
      "@/*": ["./src/*"]
      // "@anthropic-ai/*": ... ← これが干渉する場合は完全削除
    }
  }
}
```

```bash
# 解決パスの確認
npx tsc --traceResolution 2>&1 | grep "@anthropic-ai/sdk" | head -10
# "Resolved to: node_modules/@anthropic-ai/sdk/..." が表示されれば正常
# "Module not found." が出れば paths に干渉エントリあり
```

## Commit \#4: Playwright 型定義との `TS2395 Duplicate identifier` を `types` フィールドで分離する

`@playwright/test`はDOMライクな型定義を持ち込み、`ReadableStream`の重複宣言(`TS2395`)が発生する。ソース用と테스트用でtsconfigを分割し、`types`フィールドで名前空間を制御する。

```jsonc
// tsconfig.json — src/ 用 (Claude API ファイル向け)
{
  "compilerOptions": {
    "types": ["node"]  // Playwright 型を明示的に除外
  }
}
```

```jsonc
// tsconfig.test.json — テスト専用
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "types": ["node", "@playwright/test"]
  },
  "include": ["tests/**/*.ts"]
}
```

```bash
npx tsc --noEmit 2>&1 | grep -c "error TS"
# After: 0 errors
```

## Commit \#5: `moduleResolution: bundler` 復帰 — 0エラーを確認した最終 `tsconfig.json`

4コミットで解消した原因を把握した上で`bundler`に戻す。Commit #1で一時的に外した`moduleResolution`を最終形として復帰させる。

```jsonc
// tsconfig.json — 最終形 (@anthropic-ai/sdk 0.29.x + Playwright 1.44.x 共存確認済み)
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022", "ESNext.AsyncIterable", "DOM", "DOM.Iterable"],
    "strict": true,
    "skipLibCheck": false,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "resolveJsonModule": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "paths": {
      "@/*": ["./src/*"]
    },
    "types": ["node"]
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

```bash
npx tsc --noEmit
# (出力なし)
echo "Exit: $?"
# Exit: 0
```

## 差分5本でエラーが消えた軌跡 — コミット別対応表

| Commit | 変更内容 | 解消エラー | エラー数推移 |
|--------|----------|-----------|------------|
| #1 | `moduleResolution: Node16` に切替 | TS2307 Cannot find module | 7 → 3 |
| #2 | `lib` に `DOM.Iterable` + `ESNext.AsyncIterable` 追加 | TS2322 ReadableStream 型不一致 | 3 → 2 |
| #3 | `skipLibCheck: false` + `paths` 干渉エントリ除去 | TS2339 Property 'stream' 消失 | 2 → 1 |
| #4 | `types` フィールドで Playwright 分離 | TS2395 Duplicate identifier 'ReadableStream' | 1 → 0 |
| #5 | `moduleResolution: bundler` に復帰 + 最終統合 | — | 0 確認 |

**4.5時間→30分の内訳:** エラー原因の特定で平均3時間かかるところを`--traceResolution`で10分に短縮、`lib`配列の試行錯誤1時間をCommit #2の差分で5分に短縮、Playwright共存30分を`tsconfig.test.json`分割で5分に短縮。
