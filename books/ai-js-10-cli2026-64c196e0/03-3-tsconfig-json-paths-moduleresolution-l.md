---
title: "第3章：tsconfig.json paths×moduleResolutionでLangChain.js / MCP SDKが解決できない3パターン"
free: false
---

## 3エラーの診断フロー：どのパターンかを30秒で特定する

まず症状を分類する。以下のコマンドを実行して出力を照合する。

```bash
npx tsc --noEmit 2>&1 | grep -E "Cannot find module|ERR_REQUIRE_ESM|has no exported member"
```

| 出力キーワード | 該当パターン |
|---|---|
| `Cannot find module '@/` | パターン1：paths解決失敗 |
| `ERR_REQUIRE_ESM` | パターン2：ESM/CJS混在 |
| `has no exported member` + 実行時エラー | パターン3：Bundlerモード誤設定 |

---

## パターン1：`@/lib/...` が Cannot find module になる paths × baseUrl設定

**壊れる前（NG）：**

```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

`baseUrl` が未設定だと `paths` は機能しない。TypeScript 4.x系では黙って失敗し、エラーメッセージが「ファイルが存在しない」と誤誘導する。

**直す（OK）：**

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    },
    "moduleResolution": "bundler"
  }
}
```

LangChain.js 0.3+は内部で `@langchain/core/...` の再エクスポートを多用しており、`paths` の解決失敗が依存チェーン全体に伝播する。`baseUrl: "."` で解決しない場合は `rootDir` との不整合を疑う。

```bash
# rootDirとbaseUrlの不整合を診断
npx tsc --noEmit --traceResolution 2>&1 | grep "Resolving from" | head -20
```

---

## パターン2：ESM × CJS混在でMCP SDK 1.xが `ERR_REQUIRE_ESM` でクラッシュ

`@modelcontextprotocol/sdk` 1.x系はESMオンリーのパッケージ。`package.json` に `"type": "module"` がない状態で `require()` が走ると即死する。

**壊れる環境：**

```json
// tsconfig.json（CJS出力設定）
{
  "compilerOptions": {
    "module": "commonjs",
    "moduleResolution": "node"
  }
}
```

```json
// package.json（"type"フィールドなし）
{
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.3.0"
  }
}
```

**正規解法：**

```json
// tsconfig.json
{
  "compilerOptions": {
    "module": "node16",
    "moduleResolution": "node16",
    "target": "es2022"
  }
}
```

```json
// package.json
{
  "type": "module"
}
```

`node16` は `.js` 拡張子明示インポートを強制する。既存コードに `import foo from './foo'` がある場合は `'./foo.js'` への一括置換が必要。

```bash
# .js拡張子なしインポートを一括検出
grep -rE "from '\./[^']+'" src/ | grep -v "\.js'"
```

---

## パターン3：Bundlerモード × Node.js 22+ "exports" フィールド不一致

Node.js 22はパッケージの `exports` フィールドを厳密に評価する。`moduleResolution: "bundler"` はVite/esbuild前提で設計されており、`exports` の `"node"` 条件を正しく解決しないケースがある。

**症状：型チェックは通るが `node dist/index.js` で実行時エラー**

```
Error [ERR_PACKAGE_PATH_NOT_EXPORTED]: Package subpath './dist/cjs/...' is not defined by "exports"
```

**NG（Vite向けをNode.js直実行に流用）：**

```json
{
  "compilerOptions": {
    "moduleResolution": "bundler",
    "module": "esnext"
  }
}
```

**OK（Node.js 22直実行前提）：**

```json
{
  "compilerOptions": {
    "moduleResolution": "node16",
    "module": "node16",
    "target": "es2022",
    "outDir": "./dist"
  }
}
```

Viteビルドを経由する場合でも `vite.config.ts` の `ssr.target: 'node'` が抜けると同じエラーが出る。AI副業ツールのバックエンドはVite SSRではなくNode.js直実行が多数派のため、`node16` に固定した方がトラブルが少ない。

---

## jsconfig.json（JavaScript専業）向け対応表

TypeScriptを使わないプロジェクトでも `jsconfig.json` の `checkJs: true` で同等のエラーが発生する。

```json
// jsconfig.json（LangChain.js / MCP SDK 対応）
{
  "compilerOptions": {
    "module": "node16",
    "moduleResolution": "node16",
    "checkJs": true,
    "allowJs": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

| tsconfig設定 | jsconfig.json等価 | MCP SDK 1.x動作 |
|---|---|---|
| `module: "node16"` | 同一 | ○ |
| `module: "commonjs"` | 同一 | × ERR_REQUIRE_ESM |
| `moduleResolution: "bundler"` | 同一 | △ Node.js 22で一部× |
| `moduleResolution: "node"` | 同一 | × 旧API非対応 |

---

## 診断CLI：3パターンを自動判定して修正候補を出力する

本書付属の診断CLIで上記3パターンを一括チェックできる。

```bash
npx ts-node src/diagnose.ts --config tsconfig.json
```

```typescript
// src/diagnose.ts（パターン判定ロジック抜粋）
import { readFileSync } from "fs";
import { parse } from "jsonc-parser";

interface DiagResult {
  pattern: 1 | 2 | 3;
  severity: "error" | "warn";
  fix: string;
}

function diagnoseModuleResolution(configPath: string): DiagResult[] {
  const raw = readFileSync(configPath, "utf-8");
  const config = parse(raw) as { compilerOptions: Record<string, string> };
  const opts = config.compilerOptions ?? {};
  const results: DiagResult[] = [];

  // パターン1: paths × baseUrl
  if (opts.paths && !opts.baseUrl) {
    results.push({
      pattern: 1,
      severity: "error",
      fix: 'Add "baseUrl": "." to compilerOptions',
    });
  }

  // パターン2: CJS × MCP SDK
  if (opts.module === "commonjs") {
    results.push({
      pattern: 2,
      severity: "error",
      fix: 'Change "module" and "moduleResolution" to "node16"',
    });
  }

  // パターン3: Bundler × Node.js 22実行
  if (opts.moduleResolution === "bundler" && opts.module !== "esnext") {
    results.push({
      pattern: 3,
      severity: "warn",
      fix: 'Use "node16" if running directly with Node.js 22+',
    });
  }

  return results;
}

const issues = diagnoseModuleResolution(process.argv[3] ?? "tsconfig.json");
if (issues.length === 0) {
  console.log("✓ No moduleResolution issues detected");
  process.exit(0);
}
issues.forEach((r) =>
  console.log(`[Pattern ${r.pattern}][${r.severity.toUpperCase()}] ${r.fix}`)
);
process.exit(1);
```

3パターン全てに引っかかる壊れた `tsconfig.json` を用意して実行すると以下が出力される：

```
[Pattern 1][ERROR] Add "baseUrl": "." to compilerOptions
[Pattern 2][ERROR] Change "module" and "moduleResolution" to "node16"
[Pattern 3][WARN] Use "node16" if running directly with Node.js 22+
```

`package.json` の `scripts.predev` に追加しておけば `npm run dev` のたびに設定ミスを自動検出できる。第4章ではこのCLIに `--fix` フラグを追加して自動書き換えまで完成させる。
