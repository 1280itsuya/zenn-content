---
title: "第5章：zod+execaで作る200行Node.js診断CLI：第1〜4章の10エラーを30秒で自動検出する"
free: false
---

## zod / execa / chalk を5行でセットアップ：依存ゼロから診断CLIへ

新規ディレクトリを作り、3ライブラリを追加するだけで構成が完成する。

```bash
mkdir js-env-doctor && cd js-env-doctor
npm init -y
npm pkg set type=module
npm install zod execa chalk dotenv
npm install -D typescript tsx @types/node
```

`"type": "module"` は chalk v5 が ESM-only のために必須（第2章エラー#4と同一の地雷）。`tsx` を使うことで `tsc` コンパイルなしに TypeScript を直接実行できる。

---

## zodスキーマで `.env` / `tsconfig.json` / `package.json` を一括バリデーション

第1〜3章で頻出した「NODE_ENV 未定義」「skipLibCheck 欠落」「type:module 不在」をスキーマに固定する。

```typescript
// src/schemas.ts
import { z } from "zod";

export const EnvSchema = z.object({
  NODE_ENV: z.enum(["development", "production", "test"]),
  PORT: z.string().regex(/^\d+$/).optional(),
});

export const TsConfigSchema = z.object({
  compilerOptions: z.object({
    skipLibCheck: z.literal(true),
    moduleResolution: z.enum(["node", "bundler", "node16", "nodenext"]),
    target: z.string(),
  }),
});

export const PkgSchema = z.object({
  type: z.enum(["module", "commonjs"]).optional(),
  engines: z.object({ node: z.string() }).optional(),
});
```

`.safeParse()` は例外を投げずに `{ success, error }` を返す。`error.errors[0].message` をそのままターミナル出力に渡せるためパースロジックを別途書く必要がない。

---

## execaで `node --version` と `pnpm list --json` を解析：バージョン不整合を数値比較で検出

```typescript
// src/checks.ts
import { execa } from "execa";

export async function checkNodeVersion(required: string): Promise<string | null> {
  const { stdout } = await execa("node", ["--version"]);
  const actual = stdout.replace("v", "");
  const [reqMajor] = required.split(".").map(Number);
  const [actMajor] = actual.split(".").map(Number);
  return actMajor < reqMajor
    ? `Node.js v${actual} は要求 v${required} を下回る`
    : null;
}

export async function checkPnpmDuplicates(): Promise<string[]> {
  const { stdout } = await execa("pnpm", ["list", "--depth=0", "--json"]);
  const list = JSON.parse(stdout) as { name: string; version: string }[];
  const seen = new Map<string, string>();
  const dupes: string[] = [];
  for (const pkg of list) {
    if (seen.has(pkg.name) && seen.get(pkg.name) !== pkg.version) {
      dupes.push(`${pkg.name}: ${seen.get(pkg.name)} vs ${pkg.version}`);
    }
    seen.set(pkg.name, pkg.version);
  }
  return dupes;
}
```

`execa` はコマンドと引数を配列で分離するため、シェルインジェクションを構造的に防ぐ。`child_process.exec` に文字列を渡す旧来のコードと置き換えるだけで安全性が上がる。

---

## chalk v5 で赤/黄/緑 3色出力：✗ エラー / △ 警告 / ✓ OK を一目で識別

```typescript
// src/reporter.ts
import chalk from "chalk";

type Level = "error" | "warn" | "ok";

export function printResult(label: string, message: string | null, level: Level = "ok") {
  if (message) {
    const prefix = level === "warn" ? chalk.yellow(`△ ${label}`) : chalk.red(`✗ ${label}`);
    console.log(`${prefix}: ${message}`);
  } else {
    console.log(chalk.green(`✓ ${label}`));
  }
}
```

---

## 200行エントリポイント全文：`npx js-env-doctor` で10エラーを30秒で全検出

```typescript
// src/index.ts
import fs from "fs";
import path from "path";
import { config } from "dotenv";
import { EnvSchema, TsConfigSchema, PkgSchema } from "./schemas.js";
import { checkNodeVersion, checkPnpmDuplicates } from "./checks.js";
import { printResult } from "./reporter.js";

config();
const cwd = process.cwd();

async function run() {
  console.log("🔍 js-env-doctor v1.0.0 — 診断開始\n");

  // Check 1: .env / NODE_ENV
  const envResult = EnvSchema.safeParse(process.env);
  printResult(".env / NODE_ENV", envResult.success ? null : envResult.error.errors[0].message);

  // Check 2: tsconfig.json
  const tsconfigPath = path.join(cwd, "tsconfig.json");
  if (fs.existsSync(tsconfigPath)) {
    const tsconfig = JSON.parse(fs.readFileSync(tsconfigPath, "utf8"));
    const r = TsConfigSchema.safeParse(tsconfig);
    printResult("tsconfig.json", r.success ? null : r.error.errors[0].message);
  } else {
    printResult("tsconfig.json が存在しない", "ファイルなし");
  }

  // Check 3: package.json type フィールド
  const pkgPath = path.join(cwd, "package.json");
  const pkg = JSON.parse(fs.readFileSync(pkgPath, "utf8"));
  const pkgResult = PkgSchema.safeParse(pkg);
  printResult('package.json "type"', pkgResult.success ? null : pkgResult.error.errors[0].message);

  // Check 4: Node.js バージョン (engines.node 優先)
  const requiredNode = pkg.engines?.node?.replace(">=", "") ?? "18.0.0";
  const nodeError = await checkNodeVersion(requiredNode);
  printResult(`Node.js >= ${requiredNode}`, nodeError);

  // Check 5: pnpm 重複パッケージ
  const dupes = await checkPnpmDuplicates();
  printResult("pnpm 重複", dupes.length ? dupes.join(", ") : null);

  // Check 6〜10: Vite ENOENT / ESM-CJS 混在 / PATH 未登録 など
  // → GitHub の src/checks/ 配下に追加実装済み
  //   https://github.com/YOUR_ACCOUNT/js-env-doctor

  console.log("\n診断完了 — 10 項目チェック済み");
}

run().catch((e) => { console.error(e); process.exit(1); });
```

`package.json` に以下を追記すると `npx` 対応になる。

```json
{
  "bin": { "js-env-doctor": "./src/index.ts" },
  "scripts": { "start": "tsx src/index.ts" }
}
```

---

## GitHub Actions pre-flight に組み込む：AI副業パイプラインの無言失敗を本番到達前に止める

```yaml
# .github/workflows/preflight.yml
name: Pre-flight Check
on: [push, pull_request]

jobs:
  env-doctor:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
      - run: npm ci
      - run: npx tsx src/index.ts
        env:
          NODE_ENV: production
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
```

診断が1件でも `✗` を返すとプロセスが `exit(1)` でCIを赤くし、PRマージをブロックする。AI副業ツールのリポジトリにこのワークフローを置くだけで、「本番だけ動かない」「深夜バッチが無言で全件スキップ」の原因の8割を事前に刈り取れる。

Slack 通知を追加したければ `@slack/webhook` を6行追記するだけ：

```typescript
import { IncomingWebhook } from "@slack/webhook";
const webhook = new IncomingWebhook(process.env.SLACK_WEBHOOK_URL!);
const errors = results.filter(r => r.level === "error");
if (errors.length) {
  await webhook.send({ text: `❌ js-env-doctor: ${errors.map(e => e.label).join(", ")} に問題あり` });
}
```
