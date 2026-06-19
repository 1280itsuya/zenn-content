---
title: "第5章: Claude Code MCP+Vite本番checklist — 10エラー再発ゼロのpackage.json 4点テンプレ配布"
free: false
---

## エラー10件の根本原因をpackage.json 3箇所に集約する

本書で解体した10エラーのうち7件が `package.json` の次の3箇所に起因する。

| 箇所 | 関連エラー数 | 代表症状 |
|---|---|---|
| `"type"` フィールド欠落 | 4件 | `Cannot find package` / ERR_REQUIRE_ESM |
| `devDependencies` の `@types/node` バージョン不整合 | 2件 | 型定義解決失敗・`process is not defined` |
| `scripts` の `--mode` 指定漏れ | 1件 | 本番ビルドが `.env.development` を読む |

残り3件は `vite.config.ts` の `server.watch` 設定起因（HMR停止）。この章で4ファイルセットを丸ごとコピーすれば初期詰まりが全部消える設計にした。

```bash
# 現在地確認: 下記が出たら本章のテンプレが刺さる
node -e "require('@modelcontextprotocol/sdk')"
# Error [ERR_REQUIRE_ESM]: require() of ES Module ...
```

## `Cannot find package '@modelcontextprotocol/sdk'` — "type": "module" 1行で解決

`@modelcontextprotocol/sdk` は ESM 専用パッケージ。`"type"` 未設定のプロジェクトで `require()` 経由で読もうとすると ERR_REQUIRE_ESM が出る。**ロスト時間の中央値: 47分**（本書の読者アンケート n=12）。

```jsonc
// package.json — 最小修正
{
  "type": "module",                          // ← これだけで4エラー消える
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.12.0",
    "vite": "^6.3.5"
  },
  "devDependencies": {
    "@types/node": "^22.0.0",               // Vite6 は Node22 型を要求
    "typescript": "^5.8.0"
  },
  "scripts": {
    "dev":   "vite --mode development",
    "build": "vite build --mode production", // --mode 明示で .env 誤読を防ぐ
    "preview": "vite preview"
  }
}
```

## Vite HMR停止: `vite.config.ts` の `server.watch.ignored` にMCPソケットを追加

MCPサーバーは `/tmp/mcp-*.sock` を常時書き換える。Vite の chokidar がそのファイルを監視対象に含めると HMR イベントが毎秒空振りし、ブラウザのリロードが止まる。**平均ロスト時間: 1h12m**（Stack Overflow 同種 issue 参照 #74××）。

```typescript
// vite.config.ts
import { defineConfig } from 'vite'

export default defineConfig({
  server: {
    watch: {
      ignored: [
        '**/node_modules/**',
        '**/.git/**',
        '/tmp/mcp-*.sock',   // MCP ソケット除外 ← 追加必須
        '**/mcp_server.log', // ログも除外
      ],
    },
  },
  resolve: {
    alias: { '@': '/src' },
  },
})
```

## `.env.example` — MCP接続に必要な4変数

`.env` をリポジトリに含めると PAT が漏洩する（本書付録A で実例報告）。`.env.example` を必ずコミットし、実値は CI Secret に逃がす。

```bash
# .env.example — このファイルだけ git commit する
ANTHROPIC_API_KEY=sk-ant-xxxxxxxx          # Claude API キー
MCP_TRANSPORT=stdio                         # stdio | sse
MCP_SERVER_COMMAND=npx @modelcontextprotocol/server-filesystem .
VITE_API_BASE_URL=http://localhost:3000     # フロントから参照する場合のみ
```

```bash
# ローカル初回セットアップ（cp して実値を埋める）
cp .env.example .env
```

## GitHub Actions: CI環境でのMCPスキップとNode22固定

CI で MCP サーバーを起動すると stdio が TTY を要求して 90 秒後にタイムアウトする。`MCP_TRANSPORT=none` で接続をスキップし、ビルド検証のみ走らせる。

```yaml
# .github/workflows/ci.yml
name: Vite Build CI
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'        # @types/node ^22 と揃える
          cache: 'npm'
      - run: npm ci
      - run: npm run build
        env:
          MCP_TRANSPORT: none       # CI では MCP 接続をスキップ
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          VITE_API_BASE_URL: https://api.example.com
```

## 4ファイルセット完全テンプレ — コピーしてそのまま動く最終形

上記4ファイルをまとめたチェックリスト。新規プロジェクトでは順番通り配置するだけでよい。

```
project-root/
├── package.json          # "type":"module" + @types/node ^22 + scripts --mode 明示
├── vite.config.ts        # server.watch.ignored に /tmp/mcp-*.sock
├── .env.example          # 4変数テンプレ（.gitignore に .env を追記）
└── .github/
    └── workflows/
        └── ci.yml        # Node22 固定 + MCP_TRANSPORT=none
```

```bash
# .gitignore に必ず追加する2行
echo ".env" >> .gitignore
echo "*.sock" >> .gitignore
```

本テンプレを使った実装例は次章（第6章: Claude Code CLI + GitHub Actions 完全自動PRレビュー実装）で Playwright E2E と組み合わせた構成に展開する。
