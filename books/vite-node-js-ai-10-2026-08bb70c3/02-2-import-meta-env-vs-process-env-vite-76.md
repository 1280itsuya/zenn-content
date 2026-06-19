---
title: "第2章: import.meta.env vs process.env — VITE_プレフィックス誤用で76分溶かした実ログ3件"
free: false
---

## 76分の内訳：3パターンとロスト時間の実測値

このエラーで溶かした時間の内訳は次の通りだ。

| パターン | 発生箇所 | ロスト時間 |
|----------|----------|----------|
| ① `VITE_`プレフィックス欠落 | クライアント側 | 28分 |
| ② `import.meta.env`をNode.jsで呼出 | サーバーサイド | 31分 |
| ③ `.env.local`が`.gitignore`未追加 | CI/CD本番 | 17分 |

①と②は「undefinedになったAPIキーがFetchエラーを起こし、ネットワーク層のデバッグに時間を取られた」という二重罰が発生する。問題は環境変数なのに、DevToolsのNetworkタブを30分眺め続けるという典型的な誤診だ。

---

## パターン①：VITE_プレフィックス欠落でClaude APIキーがundefined（28分ロスト）

**実ログ（ブラウザコンソール）**

```
Uncaught TypeError: Cannot read properties of undefined (reading 'startsWith')
    at claudeClient.ts:12
```

コード上はこう書いていた。

```typescript
// ❌ Viteクライアントバンドルではこれはundefinedになる
const apiKey = process.env.ANTHROPIC_API_KEY;

// claudeClient.ts:12 で undefined.startsWith('sk-') がクラッシュ
const client = new Anthropic({ apiKey });
```

Viteはクライアントバンドル時に`process.env`を**静的置換しない**。`VITE_`プレフィックスを持つ変数だけが`import.meta.env`経由で公開される仕様だ（[Vite公式 #env-variables](https://vitejs.dev/guide/env-and-mode.html#env-variables)）。

**修正：`.env`ファイルとコードの両方を変更する**

```bash
# .env（リポジトリルート）
VITE_ANTHROPIC_API_KEY=sk-ant-xxxx   # VITE_プレフィックス必須
```

```typescript
// ✅ import.meta.env 経由で取得
const apiKey = import.meta.env.VITE_ANTHROPIC_API_KEY;

if (!apiKey) {
  throw new Error("VITE_ANTHROPIC_API_KEY is not defined. Check .env file.");
}
const client = new Anthropic({ apiKey });
```

`VITE_`プレフィックスなしの変数をクライアントに公開しないのはセキュリティ上正しい設計だ。逆に言えば、**秘匿すべきキーに`VITE_`を付けるとブラウザに丸見えになる**。Claude APIキーはサーバーサイドで消費すべきなので、次のパターン②が本来の正解構成になる。

---

## パターン②：Node.js側で`import.meta.env`を呼んでundefined（31分ロスト）

Viteのクライアントコードに慣れると、Expressや`ts-node`で動くサーバーサイドにも`import.meta.env`を書いてしまう。

**実ログ（Node.js起動時）**

```
ReferenceError: Cannot access 'import' before initialization
    at Object.<anonymous> (/app/server/claudeProxy.ts:3)
```

または`ts-node`の設定次第でこうなる場合もある。

```
undefined
```

つまりクラッシュすら起きず、APIキーが`undefined`のまま静かにリクエストが飛ぶ。

**問題コード**

```typescript
// server/claudeProxy.ts — Node.js (Express) で動くファイル
// ❌ Node.jsにimport.meta.envは存在しない
const apiKey = import.meta.env.ANTHROPIC_API_KEY;
```

**修正：Node.js側は`dotenv`+`process.env`に統一**

```bash
npm install dotenv
```

```typescript
// server/claudeProxy.ts
import "dotenv/config"; // ← 先頭1行で.envをprocess.envに展開

const apiKey = process.env.ANTHROPIC_API_KEY;
if (!apiKey) throw new Error("ANTHROPIC_API_KEY missing");
```

```
# .env（.gitignoreに必ず追加すること）
ANTHROPIC_API_KEY=sk-ant-xxxx
```

`import "dotenv/config"`は`require('dotenv').config()`の短縮形で、Node.js 18+ではESM importとして使える。`dotenv/config`をエントリポイント先頭に置くだけで良い。

---

## パターン③：`.env.local`を`.gitignore`未追加でCI本番キーが漏洩（17分ロスト＋セキュリティインシデント）

GitHub ActionsのログにAPIキーの先頭数文字が露出した。

**実ログ（GitHub Actions ログ）**

```
Run cat .env.local
ANTHROPIC_API_KEY=sk-ant-api03-XXXXXXXXXXXXXXXX...
```

`.env.local`を`git add -A`でコミットし、GitHub ActionsのCIがそのファイルをechoする`debug`ステップを持っていたためだ。

**即時対応コマンド（発覚から60秒以内に実行）**

```bash
# 1. Anthropicダッシュボードでキーを即時revoke
open https://console.anthropic.com/settings/keys

# 2. git履歴からファイルを完全削除（BFG使用）
brew install bfg  # または winget install --id bfg
bfg --delete-files .env.local
git reflog expire --expire=now --all && git gc --prune=now --aggressive
git push origin --force --all

# 3. .gitignoreに追加
echo ".env.local" >> .gitignore
echo ".env.*.local" >> .gitignore
git add .gitignore && git commit -m "fix: add .env.local to gitignore"
```

---

## dotenv+Vite共存の確定構成（GitHub Actions対応版）

クライアントとサーバーが同一リポジトリに混在する構成での最終解を示す。

```
project/
├── .env                  # VITE_PUBLIC_*のみ記載（クライアント公開変数）
├── .env.local            # ローカル開発のサーバー秘匿キー（.gitignore済）
├── src/                  # Viteクライアント → import.meta.env.VITE_*
└── server/               # Node.js Express → process.env（dotenv読込）
```

```typescript
// vite.config.ts — サーバー変数をクライアントに混入させない設計
import { defineConfig } from "vite";

export default defineConfig({
  // envPrefix のデフォルトは "VITE_" — 変更しない
  envPrefix: "VITE_",
});
```

---

## GitHub Actions secrets→.env マッピングYAML完全版

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Create server .env from secrets
        run: |
          cat > server/.env << EOF
          ANTHROPIC_API_KEY=${{ secrets.ANTHROPIC_API_KEY }}
          DATABASE_URL=${{ secrets.DATABASE_URL }}
          EOF
        # ↑ echoやcatで値をログに出すステップを絶対に追加しない

      - name: Create client .env for Vite build
        run: |
          echo "VITE_PUBLIC_API_BASE_URL=${{ secrets.PUBLIC_API_BASE_URL }}" > .env

      - name: Install & Build
        run: |
          npm ci
          npm run build   # ViteはBUILD時に.envを読み込む

      - name: Start server
        run: node server/index.js
        env:
          NODE_ENV: production
          # dotenv/configを使う場合、server/.envが自動読み込みされる
```

**チェックリスト（デプロイ前に必ず確認）**

```bash
# クライアントバンドルにAPIキーが混入していないか確認
grep -r "sk-ant" dist/
# → 何も出なければOK。何か出たらVITE_プレフィックスを付けた変数を即削除
```
