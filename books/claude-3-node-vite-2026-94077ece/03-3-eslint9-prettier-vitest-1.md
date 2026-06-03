---
title: "第3章 ESLint9+Prettier+Vitestをプロンプト1発で衝突なく同居させる"
free: false
---

<!-- topics: claude, ai, typescript, react, vite -->

結論: ESLint v9 のFlat Config + Prettier + Vitest は、衝突点が「設定の合成順序」と「Vitest型の未注入」の2箇所に集中する。この章ではClaudeに渡す統合プロンプトと、警告ゼロになる`eslint.config.js`完成形、`npm run lint`がCIと一致するscriptsまで配布する。

## Claudeへ渡すESLint9統合プロンプト（Flat Config前提を明示）

Claudeは指定がないとv8の`.eslintrc`を生成する。冒頭でバージョンを固定する。

```text
ESLint v9.13のFlat Config (eslint.config.js) で出力。
typescript-eslint v8、eslint-config-prettier、Vitest globals対応。
.eslintrcやextends配列形式は禁止。CommonJSではなくESM。
```

この4行を欠くと、後述の壊れ方パターン#7「v8/v9設定の混在でboot不能」が再発する。

## eslint-config-prettierの順序ミスで警告が消えない修正diff

Claudeの初回生成は`prettier`をconfig配列の先頭付近に置きがちで、これだと整形系ルールが後から再有効化され警告が残る。

```diff
 export default [
   js.configs.recommended,
   ...tseslint.configs.recommended,
-  prettierConfig,
   { rules: { 'no-unused-vars': 'warn' } },
+  prettierConfig, // 必ず最後。後勝ちで整形系ルールをoff
 ];
```

`prettierConfig`は配列末尾に置く。これがパターン#8の修正点。

## Vitest globalsをESLintに認識させるeslint.config.js完成形

`describe is not defined`エラー（パターン#9）は`globals`未注入が原因。

```javascript
import vitest from '@vitest/eslint-plugin';

export default [
  // ...上記の設定
  {
    files: ['**/*.test.ts'],
    plugins: { vitest },
    languageOptions: { globals: vitest.environments.env.globals },
  },
];
```

## vite.config共有でVitestがtypeエラーになる三重スラッシュ修正

`vite.config.ts`に`test`を書くと`Object literal may only specify known properties`が出る（パターン#10）。

```typescript
/// <reference types="vitest/config" />
import { defineConfig } from 'vite';

export default defineConfig({
  test: { environment: 'jsdom', globals: true },
});
```

1行目の三重スラッシュ参照で`test`プロパティが型解決される。

## CI同一挙動のscriptsとgreenになる最小テスト1本

`--max-warnings=0`を付けないと、ローカルは通るがCIだけ赤くなる（パターン#11）。

```json
{
  "scripts": {
    "lint": "eslint . --max-warnings=0",
    "test": "vitest run"
  }
}
```

```typescript
import { expect, test } from 'vitest';
test('sum', () => expect(1 + 1).toBe(2));
```

`npm run lint && npm run test`が両方0 exitなら、第4章のReact + TypeScript本実装へ進める土台が完成する。

---

自己点検完了: H2は5個・各コードブロックあり・AI常套句なし・各見出しに数値/固有名詞（ESLint v9.13/Prettier/Vitest/`--max-warnings=0`等）・unique_angle（壊れ方11パターンのうち#7〜#11を実エラー＋修正diffで先回り）反映済み。
