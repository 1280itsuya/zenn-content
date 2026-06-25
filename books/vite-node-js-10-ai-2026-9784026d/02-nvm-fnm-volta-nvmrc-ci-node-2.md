---
title: "nvm/fnm/Volta選定と.nvmrc地雷──CIだけ落ちるNodeバージョン不一致を2週間で完全封殺"
free: false
---

## エラー④実録：ローカル Node.js 20.11 でパスするのに GitHub Actions Node 18.x で Vite 5.2 ビルドが死ぬ

調査2日目に出た実際のエラーログがこれだ。

```
Error: Cannot find module 'vite/dist/node/chunks/dep-styfzx5a.js'
Node.js v18.19.0
  at Function.Module._resolveFilename (node:internal/modules/cjs/loader:1039:15)
```

ローカルの `node -v` は v20.11.0 を返していた。原因は `.nvmrc` を置いていたにもかかわらず、`actions/setup-node@v4` がそれを**デフォルトでは読まない**こと。

```yaml
# ❌ .nvmrc を無視する（当時の設定）
- uses: actions/setup-node@v4
  with:
    node-version: '18'

# ✅ .nvmrc を明示的に読む（修正後）
- uses: actions/setup-node@v4
  with:
    node-version-file: '.nvmrc'
```

`node-version-file` の1行を足すだけ。これで2週間のうち4日が溶けた。

---

## nvm/fnm/Volta 三択比較：Claude SDK 0.27 と Playwright 1.44 が動く環境を最速で作るなら

| ツール | Windows ネイティブ対応 | CI 連携 | 自動切替 | 速度 |
|--------|----------------------|---------|---------|------|
| nvm | △（nvm-windows は別物） | ○ | ○（.nvmrc） | 遅い |
| fnm | ○ | ○ | ○（.nvmrc/.node-version） | 速い（Rust 製） |
| Volta | ○ | ○ | ○（package.json） | 速い（Rust 製） |

`@anthropic-ai/sdk` 0.27+ は Node.js 18+ 必須、Playwright 1.44 も同様。バージョン管理の失敗はそのままツール起動失敗に直結する。

Windows 開発者が混在するチームなら **fnm 一択**。nvm-windows は本家 nvm と `.nvmrc` 互換性が低く、`--version-file-strategy local` を別途指定しないと読まれない場面がある。

```bash
# fnm インストール（Windows: winget）
winget install Schniz.fnm

# PowerShell $PROFILE に追記
fnm env --use-on-cd | Out-String | Invoke-Expression

# プロジェクトルートで即座に切替
echo "20.11.0" > .nvmrc
fnm use
```

---

## `.nvmrc` だけでは不完全 ── `package.json` の `engines` フィールドとの二重管理地雷

`.nvmrc` はバージョン管理ツールへの指示。`package.json` の `engines` フィールドは npm/pnpm への宣言。両者がずれると `pnpm install --engine-strict` で即エラーになる。

```json
// package.json
{
  "engines": {
    "node": ">=20.0.0",
    "npm": ">=10.0.0"
  }
}
```

```
// .nvmrc
20.11.0
```

Claude SDK の自動化スクリプトを pnpm で管理している場合、engines 不一致で `pnpm install` が止まる。`.nvmrc` を更新するときは必ず `engines` も同時に書き換える。

---

## エラー⑤実録：Windows で nvm を入れても Playwright install が `ENOENT node` で死ぬ

nvm-windows の既知の罠。PATH に `%APPDATA%\nvm` は追加されるが、アクティブバージョンのバイナリパス（`C:\Program Files\nodejs`）がシェル再起動後に反映されないケースがある。

```powershell
# 症状確認
nvm current   # v20.11.0 と表示
node -v       # 'node' は認識されていません ← PATH 未反映
```

```powershell
# セッション内の暫定対処
$env:PATH = "C:\Program Files\nodejs;" + $env:PATH

# 恒久対応（管理者権限 PowerShell）
[System.Environment]::SetEnvironmentVariable(
  "PATH",
  "C:\Program Files\nodejs;" + [System.Environment]::GetEnvironmentVariable("PATH","Machine"),
  "Machine"
)
```

fnm に乗り換えると PowerShell プロファイルへの1行追記で解決し、この問題は構造的に発生しない。

---

## 2週間の調査ログと最終解決 diff ── actions/setup-node matrix で Node 18/20/22 を同時検証

最終的に採用した構成。`.nvmrc`・`package.json engines`・CI matrix の3点セットで不一致を構造的に排除する。

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: ['18.x', '20.x', '22.x']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'pnpm'
      - run: corepack enable && pnpm install --frozen-lockfile
      - run: pnpm build
      - run: pnpm test
```

matrix で 18/20/22 を同時検証することで「ローカル 20 で動くが CI 18 で落ちる」問題を事前に検出できる。Claude SDK や Playwright を使うプロジェクトなら最低 Node 18/20 の2バージョンはカバーしておく。

---

## コピペ即使い：.nvmrc + package.engines + CI matrix の完全セット

```
# .nvmrc（プロジェクトルート）
20.11.0
```

```json
// package.json
{
  "engines": {
    "node": ">=20.0.0"
  },
  "packageManager": "pnpm@9.0.0"
}
```

```yaml
# .github/workflows/ci.yml（最小構成）
- uses: actions/setup-node@v4
  with:
    node-version-file: '.nvmrc'
    cache: 'pnpm'
```

この3ファイルを揃えた時点でローカル・CI 間のバージョン不一致は構造的に発生しなくなる。Windows 開発者が混在するチームでは fnm を採用し、各自の PowerShell プロファイルへの追記を README に1行明記する。Playwright の `npx playwright install --with-deps` は正しい Node が PATH に乗っていることが前提なので、この環境セットアップを PR テンプレートの必須チェック項目に入れておく。
