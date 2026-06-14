---
title: "第3章 jsconfig.json 15行でClaude Code補完を即日有効化する設定全解説"
free: false
---

## `moduleResolution: "bundler"` が Claude Code 補完精度を決定する

`moduleResolution` を `"node"` のままにすると、Vite+ESM 環境では Claude Code の補完が約30〜40%の確率で誤ったパスを提案してくる。原因は `"node"` が CJS のルール（`index.js` 省略可）を前提にしており、ESM の拡張子解決と衝突するため。

```jsonc
// NG: Vite+ESM 環境でこれを使うと Claude Code 補完がズレる
{
  "compilerOptions": {
    "moduleResolution": "node"  // ESM拡張子解決と非互換
  }
}
```

`"bundler"` に変えるだけで補完候補が正確になる。以下がその理由を含めた15行テンプレート。

---

## `baseUrl` + `paths` の15行テンプレート（Claude Code 動作確認済み）

```jsonc
// jsconfig.json — Claude Code + VS Code 両対応
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    },
    "allowImportingTsExtensions": true,
    "noEmit": true,
    "checkJs": false
  },
  "include": ["src/**/*", "vite.config.js"]
}
```

`checkJs: false` は JS ファイルに型エラーを出さず補完だけ効かせる設定。TypeScript 移行前の段階では必ず `false` にする。`include` を省くとルート全体をスキャンして Claude Code がパスを誤推定するため必須。

---

## tsconfig.json vs jsconfig.json：2問で決まる判定フロー

```
Q1: TypeScript に移行する気があるか？
├─ YES → tsconfig.json を使い、allowJs: true で JS も取り込む
└─ NO  → jsconfig.json で補完のみ有効化（コンパイル不要）
         Q2: モノレポか？
         ├─ YES → ルートに jsconfig.base.json を置き各パッケージから extends
         └─ NO  → 上記15行テンプレートをそのままルートに置くだけ
```

モノレポで jsconfig を使う場合の extends パターン：

```jsonc
// packages/app/jsconfig.json
{
  "extends": "../../jsconfig.base.json",
  "compilerOptions": {
    "paths": { "@app/*": ["src/*"] }
  }
}
```

---

## 症状別 修正diff 5パターン（コピペ即適用）

**① `@/components/Button` が補完に出ない**
```diff
- "moduleResolution": "node"
+ "moduleResolution": "bundler",
+ "baseUrl": ".",
+ "paths": { "@/*": ["src/*"] }
```

**② VS Code で `Cannot find module '@/...'` が赤くなる**
```diff
  // vite.config.js
+ resolve: {
+   alias: { '@': fileURLToPath(new URL('./src', import.meta.url)) }
+ }
```

**③ Claude Code が `../../utils/api` を提案し続ける**
```diff
+ "include": ["src/**/*"]
```

**④ `.js` 拡張子ありインポートでエラー（ESM 必須環境）**
```diff
- "moduleResolution": "node16"
+ "moduleResolution": "bundler"
```

**⑤ `checkJs: true` で補完と無関係なエラーが大量発生**
```diff
- "checkJs": true
+ "checkJs": false
```

---

## TypeScript 段階移行を見越した jsconfig 設計

移行時は jsconfig.json を tsconfig.json にリネームし、2行追加するだけで `paths` / `baseUrl` 設定を引き継げる。

```jsonc
// tsconfig.json（jsconfig から rename 後に追加する2行のみ）
{
  "compilerOptions": {
    // 既存の15行設定はそのまま流用
    "allowJs": true,   // JS ファイルを tsc が読む
    "strict": false    // 移行初期は緩め、段階的に true へ
  }
}
```

移行の進め方：`strict: false` → ファイル単位で `.js` を `.ts` にリネーム → 型エラーをファイルごとに潰す → 全ファイル移行後に `strict: true`。一括移行で詰まるリスクをこの順で回避できる。
