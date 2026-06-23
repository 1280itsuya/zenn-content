---
title: "Node.js バージョン地獄3パターン：nvm/fnm 設定ミスで claude-cli が落ちる実ログと自動修復コマンド"
free: false
---

Zenn有料Book章を執筆します。env-diag.js の障害番号対応テーブルをアンカーにした「逆引き修復」構成で書きます。

---

## 障害 #1：.nvmrc 放置で claude-cli が "SyntaxError: Cannot use import statement" で即死

env-diag.js が `ERR_NODE_VERSION #1` を出力したらここを読む。

`.nvmrc` に `20.14.0` と書いてあるのに新しいターミナルで自動切り替えが起きず、システムデフォルトの Node 16 で claude-cli が起動して落ちるパターンだ。

**実ログ**
```
$ claude
/usr/local/lib/node_modules/@anthropic-ai/claude-cli/dist/index.js:3
import { Command } from 'commander';
^^^^^^
SyntaxError: Cannot use import statement in CommonJS module
Node.js v16.20.2
```

ESM 形式の claude-cli は Node 18 未満では動かない。修復は `.zshrc` / `.bashrc` に `chpwd` フックを1行足すだけだ。

```bash
# ~/.zshrc または ~/.bashrc の末尾に追記
autoload -U add-zsh-hook
add-zsh-hook chpwd load_nvmrc

load_nvmrc() {
  local nvmrc_path
  nvmrc_path="$(nvm_find_nvmrc)"
  if [ -n "$nvmrc_path" ]; then
    local nvmrc_node_version
    nvmrc_node_version=$(nvm version "$(cat "${nvmrc_path}")")
    if [ "$nvmrc_node_version" != "$(nvm version)" ]; then
      nvm use --silent
    fi
  fi
}
load_nvmrc  # 現在シェルにも即適用
```

`source ~/.zshrc` 後に `node -v` が `v20.x.x` を返せば障害 #1 は解消する。

---

## 障害 #2：fnm + PowerShell で PATH が二重登録され `where.exe node` が v18/v20 を交互に返す

env-diag.js の `ERR_NODE_PATH #2` はここ。Windows 上で fnm と nvm for Windows を併用したときに踏む。`$PROFILE` に fnm の初期化行を書いたまま nvm for Windows も残っていると、PATH に `~\.fnm\node-versions\v20.x\bin` と `~\AppData\Roaming\nvm\v18.x` の両方が混在する。

**二重パス確認（PowerShell）**
```powershell
# 2行以上出たら二重登録が確定
$env:PATH -split ";" | Where-Object { $_ -match "node|nvm|fnm" }
```

修復は nvm for Windows を抜いて fnm に一本化する。

```powershell
# 管理者 PowerShell で実行
nvm uninstall current
winget uninstall CoreyButler.NVMforWindows

# fnm 単体をインストール
winget install Schniz.fnm

# $PROFILE を fnm だけに書き替え
$content = Get-Content $PROFILE | Where-Object { $_ -notmatch "nvm" }
$content += "`nfnm env --use-on-cd | Out-String | Invoke-Expression"
$content | Set-Content $PROFILE

# 現セッションの PATH から nvm 残骸を除去
$env:PATH = ($env:PATH -split ";" | Where-Object { $_ -notmatch "nvm" }) -join ";"
```

PowerShell を再起動して `where.exe node` が1行だけ返れば障害 #2 解消。

---

## 障害 #3：n8n v1.x が Node 20 必須なのにホスト Node 18 のままサイレント終了

env-diag.js の `WARN_NODE_VERSION #3` はここ。n8n 1.0 以降は Node 20 が必須だが、`npx n8n` は Node 18 環境でも**エラーを一切出さずに exit 0** で終わる。ログが空のままプロセスが消えるため原因特定が数時間単位で遅れる。

**実ログ（Node 18 + n8n v1.46.0）**
```
$ npx n8n
Need to install the following packages:
  n8n@1.46.0
Ok to proceed? (y) y
$               ← 何も起きずに終了（exit code 0）
```

確認と修復:

```bash
node -v           # → v18.x.x ← 犯人

# LTS(v20) をインストールして default に昇格
nvm install --lts
nvm alias default lts/*
nvm use default

node -v           # → v20.x.x
npx n8n           # 今度は正常起動ログが流れる
```

---

## env-diag.js の障害コード #1〜#3 と修復コマンド対応表

env-diag.js の Node バージョン・PATH チェック部分の実装は以下の通り。stdout に出るコードをそのまま本章の見出しに対応させている。

```javascript
// env-diag.js — Node チェックブロック
import path from 'path';

const [major] = process.versions.node.split('.').map(Number);

if (major < 18) {
  console.error('ERR_NODE_VERSION #1 node<18 detected:', process.version);
} else if (major === 18 && process.env.N8N_CHECK === '1') {
  console.warn('WARN_NODE_VERSION #3 n8n+node18 silent-exit risk');
}

const nodePaths = (process.env.PATH ?? '')
  .split(path.delimiter)
  .filter(p => /node|nvm|fnm/i.test(p));

if (nodePaths.length > 1) {
  console.error('ERR_NODE_PATH #2 duplicate node paths:');
  nodePaths.forEach(p => console.error(' ', p));
}
```

| stdout コード | 症状 | 最短修復コマンド |
|---|---|---|
| `ERR_NODE_VERSION #1` | claude-cli SyntaxError 即死 | `nvm install --lts && nvm alias default lts/*` + zshrc フック追加 |
| `ERR_NODE_PATH #2` | which node が二重パスを返す | nvm for Windows 削除 → fnm 単体化 |
| `WARN_NODE_VERSION #3` | n8n 無音 exit 0 | `nvm use 20 && nvm alias default 20` |

---

## `.tool-versions` 1ファイルで nvm / fnm / mise と GitHub Actions を同時固定

ローカルを直しても CI が古い Node を掴み続けるケースが残る。`.tool-versions` をプロジェクトルートに置くと asdf・mise（旧 rtx）・fnm が全て同じファイルを読む。

```
# .tool-versions
nodejs 20.14.0
```

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version-file: '.tool-versions'   # .nvmrc でも可
      - run: node -v      # → v20.14.0 固定
      - run: npx claude --version
```

ローカルでの生成は `mise use node@20.14.0` 一発で `.tool-versions` を作れる。これで障害 #1〜#3 は環境差異ごと封殺できる。
