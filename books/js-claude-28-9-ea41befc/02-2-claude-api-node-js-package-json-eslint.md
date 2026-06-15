---
title: "第2章｜Claude API + Node.js で package.json・ESLint・Prettier を30秒自動生成：全コードと6つのつまずきエラーログ"
free: false
---

## Claude API を Node.js から呼び出す最小実装：claude-sonnet-4-6 + 分割生成で JSON 切れを防ぐ

章末の `gen-config.js` を `node gen-config.js react spa` で実行すると、`package.json` / `.eslintrc.js` / `.prettierrc.json` / `tsconfig.json` の4ファイルが30秒以内に生成される。ただし「1回のAPIコールで全ファイルをまとめて生成する」実装は **エラー①（トークン超過によるJSON破損）** で必ず詰まる。最初から分割生成にしておく。

```javascript
// gen-config.js（ES Modules形式）
import Anthropic from "@anthropic-ai/sdk";
import fs from "fs/promises";
import path from "path";
import { buildPrompt } from "./prompts.js";
import { extractJSON, dedupePackageJson, writeFile } from "./utils.js";

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

const FRAMEWORK = process.argv[2] ?? "react"; // react | vue | next | vanilla
const PURPOSE   = process.argv[3] ?? "spa";   // spa | api | shopify

const FILES = ["package.json", ".eslintrc.js", ".prettierrc.json", "tsconfig.json"];

(async () => {
  for (const file of FILES) {
    const msg = await client.messages.create({
      model: "claude-sonnet-4-6",
      max_tokens: 1024,           // 1ファイルならこれで十分
      messages: [{ role: "user", content: buildPrompt(file, FRAMEWORK, PURPOSE) }],
    });
    let content = extractJSON(msg.content[0].text);
    if (file === "package.json") content = dedupePackageJson(content);
    await writeFile(file, content);
  }
  console.log("✅ 4ファイル生成完了");
})();
```

```bash
ANTHROPIC_API_KEY=sk-ant-... node gen-config.js next shopify
```

---

## React/Next.js/Shopify 別プロンプトテンプレート：依存パッケージを引数1つで動的切替

```javascript
// prompts.js
const FRAMEWORK_DEPS = {
  react:   '"react": "^18.3.0", "react-dom": "^18.3.0"',
  vue:     '"vue": "^3.4.0"',
  next:    '"next": "^14.2.0", "react": "^18.3.0", "react-dom": "^18.3.0"',
  vanilla: '""',
};

const PURPOSE_DEPS = {
  spa:     '"vite": "^5.4.0", "@vitejs/plugin-react": "^4.3.0"',
  api:     '"express": "^4.21.0", "@types/express": "^5.0.0"',
  shopify: '"@shopify/cli": "^3.67.0", "@shopify/polaris": "^12.0.0"',
};

export function buildPrompt(file, framework, purpose) {
  return `
${file} を生成してください。JSONのみ返す。説明文・コードフェンス不要。

フレームワーク: ${framework}
用途: ${purpose}
主要依存: ${FRAMEWORK_DEPS[framework]}, ${PURPOSE_DEPS[purpose]}
TypeScript: true

制約:
- ESLint は v8 系・.eslintrc.js（CommonJS）形式のみ。flat config(eslint.config.js)は生成しない
- ESLint extends の末尾に必ず "prettier" を追加
- package.json の scripts.build は CI が呼ぶビルドコマンドにする
- @types/* は devDependencies にのみ入れる（dependencies には絶対に入れない）
`.trim();
}
```

---

## extractJSON と dedupePackageJson：マークダウン混入と重複キーを自動除去

```javascript
// utils.js
import fs from "fs/promises";

export function extractJSON(text) {
  // コードフェンスが混入した場合に除去
  const stripped = text
    .replace(/^```(?:json)?\n?/m, "")
    .replace(/\n?```$/m, "")
    .trim();

  try {
    return JSON.parse(stripped);
  } catch {
    // JSON 末尾が切れた場合の応急処置（エラー①対策）
    const recovered = stripped.replace(/,?\s*$/, "") + "}";
    return JSON.parse(recovered);
  }
}

// @types/* が dependencies と devDependencies の両方に現れる場合に除去（エラー④対策）
export function dedupePackageJson(pkg) {
  const deps    = { ...(pkg.dependencies    ?? {}) };
  const devDeps = { ...(pkg.devDependencies ?? {}) };
  for (const key of Object.keys(deps)) {
    if (key in devDeps) delete devDeps[key];
  }
  return { ...pkg, dependencies: deps, devDependencies: devDeps };
}

export async function writeFile(filename, content) {
  const text = typeof content === "string"
    ? content
    : JSON.stringify(content, null, 2);
  await fs.writeFile(filename, text, "utf-8");
  console.log(`  → ${filename}`);
}
```

---

## エラー①〜③：74分溶けた JSON 破損・ESLint v9 flat config 未対応・peer dependency 競合

**エラー① トークン超過によるJSON破損（ロスト: 18分）**

4ファイルを一括生成すると `max_tokens: 4096` でも tsconfig が大きい場合に切れる。

```
SyntaxError: Unexpected end of JSON input
    at JSON.parse (<anonymous>)
```

`extractJSON` の応急処置でも回収できない場合は、そのファイルだけ再取得する。前掲の分割生成ループに変えた時点でほぼ発生しなくなった。

---

**エラー② ESLint v9 flat config 未対応（ロスト: 34分）**

Claude がデフォルトで `eslint.config.js`（flat config）を生成する。プロジェクトに `.eslintrc.js` と `eslint.config.js` が混在すると ESLint v9 は `.eslintrc.js` を無視する。全ルールが無効になるため lint が素通りし、CI に入れた後で発覚した。プロンプトへの明示追加で再発ゼロ。

```javascript
// prompts.js の制約欄に追加した一文
`ESLint は v8 系・.eslintrc.js（CommonJS）形式のみ。flat config(eslint.config.js)は生成しない`
```

---

**エラー③ peer dependency 競合（ロスト: 22分）**

Shopify 用途で `@shopify/polaris@12` と `react@18` の peer が食い違う。

```
npm warn ERESOLVE overriding peer dependency
npm error peer react@"^17.0.0" from @shopify/polaris@11.x
```

```bash
# 生成された package.json をそのまま npm install しない
# polaris のバージョンを先に固定してから install する
npm install @shopify/polaris@12 --save-exact
npm install
```

---

## エラー④〜⑥：64分溶けた 型定義重複・Prettier/ESLint 競合・npmスクリプト衝突

**エラー④ @types/* 重複（ロスト: 24分）**

Claude が `@types/react` を `dependencies` と `devDependencies` 両方に出力するケースが7回中3回あった。Node.js は後者でキーを上書きするためバージョン不整合になる。前掲の `dedupePackageJson` で自動除去。

---

**エラー⑤ Prettier vs ESLint ルール競合（ロスト: 22分）**

```
  1:1  error  Expected indentation of 2 spaces but found 0  indent
```

`eslint-plugin-prettier` を入れたのに `extends` の末尾に `"prettier"` がなく、`indent` ルールが Prettier と競合。プロンプトに制約を追加して解決。

```javascript
// .eslintrc.js の正しい extends
module.exports = {
  extends: [
    "eslint:recommended",
    "plugin:react/recommended",
    "plugin:@typescript-eslint/recommended",
    "prettier",                    // ← 必ず末尾
  ],
};
```

---

**エラー⑥ npm scripts.build の衝突（ロスト: 18分）**

Shopify 用途で Claude が `"build": "shopify theme push"` を生成した。Vercel / GitHub Actions の `npm run build` は exit 0 で偽の成功を返し、本番デプロイが shopify push になった。

```json
// 修正後（shopify 用途）
{
  "scripts": {
    "dev":    "shopify theme dev",
    "build":  "shopify theme check",
    "push":   "shopify theme push",
    "deploy": "shopify theme push --allow-live"
  }
}
```

プロンプトの制約行 `scripts.build は CI が呼ぶビルドコマンドにする` で再発防止。

---

## 実測ログ：23案件の初動時間と API コスト（2026年5〜6月）

| 案件タイプ | 手動（分） | gen-config（分） | 短縮率 | API コスト |
|-----------|----------:|---------------:|------:|----------:|
| React SPA | 62 | 28 | 55% | ¥4 |
| Next.js + API Routes | 89 | 43 | 52% | ¥7 |
| Shopify テーマ | 118 | 67 | 43% | ¥12 |
| Vanilla + Express | 55 | 28 | 49% | ¥3 |

エラーログを踏まない前提でも Shopify で **43%短縮**。6つのエラーを自力解決しながら進んでいた初期は実質 +2時間18分 が乗っていたので、そこを省けるのが本章の核心。

```
gen-config/
├── gen-config.js   # メインスクリプト（本章掲載コード）
├── prompts.js      # buildPrompt
├── utils.js        # extractJSON / dedupePackageJson / writeFile
└── .env.example    # ANTHROPIC_API_KEY=
```

```bash
git clone https://github.com/your-handle/gen-config
cd gen-config && npm install
cp .env.example .env   # ANTHROPIC_API_KEY を設定
node gen-config.js next api
```
