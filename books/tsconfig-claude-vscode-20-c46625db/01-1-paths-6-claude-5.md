---
title: "第1章 paths解決失敗6パターンをClaudeで5分修正するプロンプト"
free: true
---

## topics: ["typescript", "nextjs", "claude", "vscode", "monorepo"]

結論から言うと、`Cannot find module '@/...'` の9割は `tsconfig.json` の `baseUrl` と `paths` の2行で直る。本章では Next.js / Vite で実際に詰まった6パターンを、エラー文→Claudeプロンプト→検証コマンドの順で潰す。

## パターン1: baseUrl 未設定で @/ が全滅するエラー

```bash
error TS2307: Cannot find module '@/lib/db' or its corresponding type declarations.
```

Claudeにこのプロンプトを貼る（tsconfig.json全文も一緒に貼る）。

```text
次のtsconfig.jsonとエラー文を見て、原因を1行で示し、
修正後のcompilerOptionsをdiff形式だけで返して。解説は不要。
[エラー文] error TS2307: Cannot find module '@/lib/db'
[tsconfig.json] ←ここに全文貼る
```

返ってくる修正diff：

```diff
   "compilerOptions": {
+    "baseUrl": ".",
+    "paths": { "@/*": ["src/*"] }
   }
```

## パターン2: paths は書いたのに Vite だけ解決しないエラー

`vite.config.ts` 側のalias欠落が原因。`vite-tsconfig-paths` を入れる。

```bash
npm i -D vite-tsconfig-paths
```

```ts
// vite.config.ts
import tsconfigPaths from 'vite-tsconfig-paths'
export default { plugins: [tsconfigPaths()] }
```

## パターン3: monorepo で @/ が別パッケージを指すエラー

ルートと各workspaceで `baseUrl` が衝突する典型。各パッケージ側 tsconfig に明示する。

```jsonc
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": { "baseUrl": ".", "paths": { "@/*": ["./src/*"] } }
}
```

## パターン4: VSCodeだけ赤線が消えないエラー

tsc は通るのにエディタが追従しない。コマンドパレットで TS Server を再起動する。

```text
Cmd/Ctrl+Shift+P → "TypeScript: Restart TS Server"
```

`.vscode/settings.json` でワークスペースのTSを固定する。

```json
{ "typescript.tsdk": "node_modules/typescript/lib" }
```

## パターン5・6を直したら必ず実行する検証コマンド

修正diffを当てたら、型チェックとビルドの2段で確認する。これを通さない修正は信用しない。

```bash
npx tsc --noEmit          # 型解決だけ検証（数秒）
npx next build            # 実ビルドで paths を最終確認
```

`error TS` がゼロ行で終われば完了。ここまでで `@/` エイリアス崩壊は5分で直る。

残る5パターン（`rootDir` 衝突・`include` 漏れ・`composite` プロジェクト参照ほか）と、実際に詰まった**tsconfigエラー47件のログ＋検証済みプロンプト20選**は第2章以降に収録した。同じ `error TS2307` でも原因は6通りあり、貼るtsconfigの範囲を1行間違えるとClaudeは別の修正を返す——その切り分け表を次章で配る。
