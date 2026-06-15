---
title: "第3章 .envファイル未ロードでCLAUDE_API_KEYがundefinedになる5つの落とし穴と修正パターン"
free: false
---

frontmatter に topics を追加しつつ、5シナリオすべてに修正前後のコード diff を対応付けた章を書きます。

---

```markdown
---
topics: ["typescript", "claude", "nodejs", "automation", "javascript"]
---

# 第3章 .envファイル未ロードでCLAUDE_API_KEYがundefinedになる5つの落とし穴と修正パターン

第1章の診断スクリプトが `CLAUDE_API_KEY: ❌ undefined` を返したとき、原因は「.envファイルが存在しない」ではなくほぼ確実に「存在するのに読まれていない」。5つのシナリオ別に修正パターンを示す。

---

## 落とし穴①: `require('dotenv').config()` より先にimportを書くとprocess.envが空になる

**NG例 — claude-client.ts でキーが取れない**

Node.js のモジュール評価は import/require の解決時点で実行される。`dotenv.config()` より先に `claudeClient` をimportすると、そのファイルの `process.env.CLAUDE_API_KEY` は `undefined` のまま束縛される。

```typescript
// ❌ NG: claudeClient が評価される時点でまだ .env が読まれていない
import { claudeClient } from './claude-client';
import dotenv from 'dotenv';
dotenv.config();
```

```typescript
// ✅ OK: dotenv を最初に実行してから他をimport
import dotenv from 'dotenv';
dotenv.config(); // ← 必ずトップレベルの最初の行

import { claudeClient } from './claude-client';
```

診断スクリプトのNG出力例:
```
[diag] dotenv.config() call order: AFTER first import → RISK
[diag] CLAUDE_API_KEY at import time: undefined
```

ESM (`import` 構文) を使う場合は `dotenv/config` をサイドエフェクトインポートするか、`--require dotenv/config` フラグを使う。

```bash
# ESM プロジェクトでのエントリポイント実行
node --require dotenv/config dist/index.js
```

---

## 落とし穴②: TypeScript で `process.env.CLAUDE_API_KEY` が `string | undefined` のまま型エラーになりビルドが通らない

厳格な TypeScript プロジェクトでは `noUncheckedIndexedAccess` や `strictNullChecks` が有効なため、`undefined` 混じりの変数をそのまま渡すとコンパイルエラーになる。

```typescript
// ❌ NG: Argument of type 'string | undefined' is not assignable to parameter of type 'string'
const client = new Anthropic({ apiKey: process.env.CLAUDE_API_KEY });
```

修正は「アサーション」か「明示的なバリデーション」の2択。副業ツールでは起動時に即クラッシュさせるバリデーション型が安全。

```typescript
// ✅ OK: 起動時にvalidateして存在が保証された変数を使う
function requireEnv(key: string): string {
  const val = process.env[key];
  if (!val) throw new Error(`Missing required env var: ${key}`);
  return val;
}

const apiKey = requireEnv('CLAUDE_API_KEY'); // string 型が確定
const client = new Anthropic({ apiKey });
```

型定義ファイルで `process.env` を拡張する方法もあるが、実体チェックを省略できてしまうため副業ツールには向かない。

```typescript
// src/env.d.ts — 型補完用途のみ (validateは別途必須)
declare namespace NodeJS {
  interface ProcessEnv {
    CLAUDE_API_KEY: string;
    NODE_ENV: 'development' | 'production';
  }
}
```

---

## 落とし穴③: GitHub Actions で `env:` ハードコード値が `secrets.CLAUDE_API_KEY` を上書きする

`env:` コンテキストに同名キーを書くと、`secrets:` より優先されてハードコード値が渡る。エラーが出ず `CLAUDE_API_KEY=your-key-here` のまま実行が続くため発見が遅れる。

```yaml
# ❌ NG: env: の文字列が secrets より優先される
jobs:
  run:
    runs-on: ubuntu-latest
    env:
      CLAUDE_API_KEY: "your-key-here"   # ← これが secrets を上書き
    steps:
      - run: node dist/index.js
        env:
          CLAUDE_API_KEY: ${{ secrets.CLAUDE_API_KEY }}
```

```yaml
# ✅ OK: job レベルの env: から同名キーを削除し、step レベルのみで注入
jobs:
  run:
    runs-on: ubuntu-latest
    # job レベルに CLAUDE_API_KEY を書かない
    steps:
      - uses: actions/checkout@v4
      - run: node dist/index.js
        env:
          CLAUDE_API_KEY: ${{ secrets.CLAUDE_API_KEY }}
```

確認コマンド:
```bash
# GitHub CLI でシークレット一覧と設定状況を確認
gh secret list --repo your-org/your-repo
```

---

## 落とし穴④: Docker Compose の `environment:` セクションが `.env` ファイルの値を空文字で上書きする

`docker compose up` は `.env` ファイルを自動読み込みするが、`environment:` セクションに同名キーを空で定義すると空文字が勝つ。

```yaml
# ❌ NG: environment: の空定義が .env の値を消す
services:
  bot:
    build: .
    environment:
      - CLAUDE_API_KEY=          # ← 空文字で上書き
      - NODE_ENV=production
```

```yaml
# ✅ OK: .env から引き継ぐキーは値なしで宣言するか、environment: に書かない
services:
  bot:
    build: .
    env_file:
      - .env                     # ← 明示的に env_file で読む
    environment:
      - NODE_ENV=production      # 上書きしたいキーだけ書く
```

実行時確認:
```bash
# コンテナ内の環境変数を確認
docker compose run --rm bot env | grep CLAUDE
```

---

## 落とし穴⑤: pnpm ワークスペースでルートの `.env` が子パッケージに届かない

pnpm ワークスペース構成では、各パッケージの `dotenv.config()` はそのパッケージの `cwd` を基準に `.env` を探す。ルートにだけ `.env` を置くと子パッケージからは見えない。

```
monorepo/
├── .env                 ← ルートに CLAUDE_API_KEY を書いた
├── packages/
│   └── claude-bot/
│       └── src/index.ts ← dotenv.config() がここの cwd を参照
```

```typescript
// ❌ NG: cwd が packages/claude-bot/ になるためルートの .env を読まない
import dotenv from 'dotenv';
dotenv.config();
```

```typescript
// ✅ OK: __dirname から相対パスでルートの .env を指定
import dotenv from 'dotenv';
import path from 'path';

dotenv.config({
  path: path.resolve(__dirname, '../../../.env'), // ← モノレポルートを明示
});
```

または、ルートに `.env` を置かず各パッケージにシンボリックリンクを張る方法も有効。

```bash
# packages/claude-bot/ 内でルートの .env をリンク
cd packages/claude-bot
ln -s ../../.env .env
```

診断スクリプトのNG出力例:
```
[diag] dotenv search path: /monorepo/packages/claude-bot/.env
[diag] file exists: false
[diag] CLAUDE_API_KEY: undefined
[diag] hint: run with --root-env flag or set path option
```

---

## 5シナリオの判別フローチャート

```
CLAUDE_API_KEY === undefined?
│
├─ ESM/CJSのimport順序を確認 → 落とし穴①
├─ TypeScriptビルドエラー?  → 落とし穴②
├─ GitHub Actions上だけ失敗? → 落とし穴③
├─ Docker内だけ失敗?        → 落とし穴④
└─ pnpmワークスペース構成?   → 落とし穴⑤
```

第1章の診断スクリプトに `--scenario` フラグを渡すと、上記フローに従って該当シナリオを自動判定する。次章では、APIキーが正しく渡っているのにClaude APIから `401 Unauthorized` が返るケースを扱う。
```

---

frontmatter に `topics: ["typescript", "claude", "nodejs", "automation", "javascript"]` を追加し、5シナリオそれぞれに修正前後のコード diff と診断スクリプト出力例を対応付けました。末尾のフローチャートで「自分の症状→該当落とし穴」への逆引き動線も確保しています。
