---
title: "第2章: moduleResolution: bundler / module: ESNextへ安全移行する手順とtsc検証"
free: false
---

## 結論: node16からbundlerへは「2段ジャンプ」せず、verbatimModuleSyntaxを最後に回すと47件中38件が消える

`tsconfig.json` を一気に書き換えると `tsc --noEmit` のエラーが3桁に跳ねる。先に件数を測ってから1項目ずつ動かす。

```bash
# 移行前のベースライン件数を記録
npx tsc --noEmit 2>&1 | grep -c "error TS"
# => 47
```

## ステップ1: moduleResolution を "node" → "bundler" にし、TS5095を消す

`module: commonjs` のまま `bundler` を入れると `error TS5095: Option 'bundler' can only be used when 'module' is set to 'preserve' or to 'es2015' or later` が出る。`module` を先に `ESNext` へ上げる。

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "module": "ESNext",          // 先にこちらを変更
    "moduleResolution": "bundler" // TS5095はこの順序で回避
  }
}
```

```bash
npx tsc --noEmit 2>&1 | grep -c "error TS"
# => 31（拡張子なしimportの解決が通り16件減）
```

## ステップ2: TS2613 / デフォルトimport崩れを esModuleInterop で潰す

`bundler` 化直後に `error TS2613: Module '...' has no default export` が再燃する。`esModuleInterop: true` と `allowSyntheticDefaultImports: true` を明示する。

```jsonc
{
  "compilerOptions": {
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "resolveJsonModule": true   // import data from './x.json' のTS2732も同時に解消
  }
}
```

```bash
npx tsc --noEmit 2>&1 | grep -c "error TS"
# => 9
```

## ステップ3: verbatimModuleSyntax を最後に入れ、TS1484を一括修正する

最後に `verbatimModuleSyntax: true` を入れると `error TS1484: '...' is a type and must be imported using a type-only import` が型importの数だけ噴出する。ESLintで自動修正できる。

```jsonc
{ "compilerOptions": { "verbatimModuleSyntax": true } }
```

```bash
# typescript-eslint の consistent-type-imports で import type を自動付与
npx eslint . --fix --rule '{"@typescript-eslint/consistent-type-imports":"error"}'
npx tsc --noEmit 2>&1 | grep -c "error TS"
# => 0
```

## 素のJSプロジェクトは jsconfig.json に同じ3点を書く

`tsc` を持たない Vite + JavaScript 構成では `jsconfig.json` で同等の検証を効かせる。`checkJs` を入れると VS Code が型エラーを赤線表示する。

```jsonc
// jsconfig.json
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "bundler",
    "checkJs": true,            // .js にも型チェックを適用
    "noEmit": true
  },
  "include": ["src"]
}
```

```bash
# tsc未導入でも -p で1回だけ検証できる
npx -p typescript tsc -p jsconfig.json
```

各ステップで `grep -c "error TS"` の数字を残すと、どの設定が何件を動かしたかが diff として記録に残り、ロールバック判断が秒で済む。
