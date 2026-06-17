---
title: "第4章：Vitest ESM設定とJest transformIgnorePatterns地雷を根絶する設定ファイル全掲載"
free: false
---

## Jest 29で`SyntaxError: Cannot use import statement`が出る構造的原因

Jest 29はデフォルトでCommonJS変換を前提とし、`node_modules`内ファイルを`transformIgnorePatterns`でBabel変換からバイパスする。`openai` v4・`@anthropic-ai/sdk` v0.20+・`chalk` v5・`node-fetch` v3はESM-onlyのため、デフォルト設定で実行すると必ず以下のエラーが出る。

```
SyntaxError: Cannot use import statement outside a module
  at Runtime.createScriptFromCode (node_modules/jest-runtime/build/index.js:1796:14)
  at Object.<anonymous> (node_modules/openai/index.js:1:1)
```

デフォルトの`transformIgnorePatterns`は`["/node_modules/"]`で全`node_modules`を変換スキップする。修正は「対象パッケージだけをスキップから除外」する否定先読み正規表現1行のみ。

## `transformIgnorePatterns` 正規表現1行：OpenAI/Anthropic SDK対応版

```typescript
// jest.config.ts
import type { Config } from 'jest';

const config: Config = {
  preset: 'ts-jest/presets/default-esm',
  testEnvironment: 'node',
  extensionsToTreatAsEsm: ['.ts'],
  transformIgnorePatterns: [
    // 括弧内パッケージのみBabel変換対象、それ以外のnode_modulesはスキップ
    '/node_modules/(?!(openai|@anthropic-ai|chalk|node-fetch|@sindresorhus|got)/).*',
  ],
  moduleNameMapper: {
    // ESMの.js拡張子をJestが解決できない問題を回避
    '^(\\.{1,2}/.*)\\.js$': '$1',
  },
};

export default config;
```

スコープ付きパッケージ名（`@anthropic-ai`）は`/`ごと括弧内に記述する。パイプ`|`でパッケージを追加するだけで対応範囲を拡張できる。

## Vitest移行：`vite.config.ts` 最小構成とglobals設定

VitestはViteのESMバンドラーを使うため、ESM-onlyパッケージで詰まる頻度がJestと比べて大幅に減る。移行時の最小構成は以下。

```typescript
// vite.config.ts
import path from 'path';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  test: {
    globals: true,        // describe/it/expectをimport不要で使う
    environment: 'node',
    setupFiles: ['./vitest.setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json'],
    },
  },
});
```

```bash
# 旧Jest依存を削除してVitestに置き換える
npm remove jest ts-jest @types/jest babel-jest
npm install -D vitest @vitest/coverage-v8
```

Jestの`moduleNameMapper`で`.js`拡張子を再マッピングしていたルール（`'^(\\.{1,2}/.*)\\.js$': '$1'`）はVitest側では不要なため削除してよい。

## OpenAI/Anthropic SDKの`fetch/global.fetch`モックが壊れる問題

`openai` v4と`@anthropic-ai/sdk`はNode.js 18+のグローバル`fetch`を前提とする。Node.js 16以下またはJestのjsdom環境では`fetch is not defined`が出る。

```typescript
// vitest.setup.ts（Node.js 16以下のCI環境向けpolyfill）
import { fetch, Headers, Request, Response } from 'undici';

Object.assign(global, { fetch, Headers, Request, Response });
```

```bash
npm install -D undici
```

Node.js 18以上をCIで固定すればpolyfillは不要だが、`.nvmrc`や`package.json`の`engines`フィールドで固定していない場合に踏む。`.nvmrc`に`18.20.4`と記述するだけで根絶できる。

## Vitestで`vi.hoisted()`を使ったAnthropicSDKモックの正解パターン

ESM環境でJestのdynamic mockを移植すると`Cannot mock module before it's imported`が出る。Vitest 0.31+の`vi.hoisted()`で回避する。

```typescript
// src/llm/__tests__/anthropic.test.ts
import { vi, describe, it, expect, beforeEach } from 'vitest';

// vi.hoisted()でモックファクトリを定義 → vi.mock()巻き上げと初期化順序の不一致を回避
const mockCreate = vi.hoisted(() => vi.fn());

vi.mock('@anthropic-ai/sdk', () => ({
  default: vi.fn().mockImplementation(() => ({
    messages: {
      create: mockCreate,
    },
  })),
  Anthropic: vi.fn().mockImplementation(() => ({
    messages: {
      create: mockCreate,
    },
  })),
}));

describe('Anthropic messages.create', () => {
  beforeEach(() => {
    mockCreate.mockReset();
  });

  it('正常レスポンスを返すこと', async () => {
    mockCreate.mockResolvedValueOnce({
      content: [{ type: 'text', text: 'テスト応答' }],
      usage: { input_tokens: 10, output_tokens: 5 },
    });

    // テスト対象コードを呼ぶ
    const { callClaude } = await import('../anthropic');
    const result = await callClaude('テストプロンプト');

    expect(mockCreate).toHaveBeenCalledOnce();
    expect(mockCreate).toHaveBeenCalledWith(
      expect.objectContaining({ model: expect.stringContaining('claude') })
    );
    expect(result).toBe('テスト応答');
  });
});
```

## テスト環境固有バグ5件：エラーコード別修正早見表

| # | エラー文 | 原因 | 1行修正 |
|---|---------|------|--------|
| 1 | `SyntaxError: Cannot use import statement` | ESM-onlyパッケージがJest変換スキップ | `transformIgnorePatterns`に否定先読み正規表現追加 |
| 2 | `TypeError: fetch is not defined` | Node.js 16以下/jsdom環境 | `undici` polyfill + `setupFiles`登録 |
| 3 | `Cannot find module '...' with '.js'` | Jestが`.js`拡張子のESMパスを解決不可 | `moduleNameMapper`で`.js`→拡張子なしに変換 |
| 4 | `ReferenceError: describe is not defined` | Vitestで`globals: true`未設定 | `vite.config.ts`の`test`に`globals: true`追加 |
| 5 | `Cannot mock module before it's imported` | ESM環境でのモック宣言順序問題 | `vi.hoisted()`でモックファクトリを先頭定義 |

バグ3の`moduleNameMapper`はJestのみの問題。Vitestでは`resolve.alias`だけ設定すれば`.js`拡張子も自動解決するため、Vitest移行だけでバグ1・3・5が同時に消えるケースが多い。移行コストが低いプロジェクトではVitest一択を検討する価値がある。
