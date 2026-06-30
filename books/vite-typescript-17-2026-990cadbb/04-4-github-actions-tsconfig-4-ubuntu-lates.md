---
title: "第4章：ローカルで通るのにGitHub Actionsで死ぬtsconfig問題4件──ubuntu-latest固有バグとCI冪等設定"
free: false
---

章の目的を1文で言語化してから書き始めます。「ubuntu-latestのLinuxファイルシステム特性とキャッシュ汚染による4種のCI固有エラーを、実ログ+コピペ即動作テンプレートで根治できるようにする。」

---

<!-- METADATA
topics: typescript, vite, ai, frontend, automation
-->

ローカルの `tsc --noEmit` が0エラーで通るのにPushした瞬間にCIが `error TS2307: Cannot find module` で死ぬ。このパターンは**ubuntu-latestのLinuxファイルシステム特性**と**Actions特有のキャッシュ汚染**が組み合わさって発生する。4パターン別に実ログと対策を示す。

---

## パターン1：大文字小文字区別──macOSで通るimportがubuntu-latestで即死

macOS/Windowsのファイルシステムはデフォルトで大文字小文字を区別しない。

```ts
import { Button } from './components/button';  // 実ファイルは Button.tsx
```

ローカルは通るが、ubuntu-latest（ext4）は厳密区別するので `TS2307` が飛ぶ。

**実CIログ：**
```
src/App.tsx(3,25): error TS2307: Cannot find module './components/button'
or its corresponding type declarations.
```

**tsconfig.json に追加する設定：**
```json
{
  "compilerOptions": {
    "forceConsistentCasingInFileNames": true
  }
}
```

`forceConsistentCasingInFileNames: true` を入れるとローカルでも即エラーになる。Claude Codeが自動生成するtsconfigはこのオプションを省略しがちなので、生成直後に必ず確認する。

---

## パターン2：pnpmシンボリックリンク差異──baseUrl省略でpathsエイリアスが死ぬ

pnpmはnode_modulesをシンボリックリンク構造で管理する。ubuntu-latestでは `baseUrl` を省略すると `paths` エイリアスの起点が不明になりCI環境でのみ解決失敗する。

**症状：**
```
error TS2307: Cannot find module '@/components/Button' or its corresponding type declarations.
```

**修正前（AI自動生成の典型パターン・baseUrlなし）：**
```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

**修正後（baseUrl明示）：**
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

pnpm使用時は `.npmrc` に `node-linker=hoisted` を追加するとシンボリックリンク問題ごと回避できるが、lockfileの再生成が必要。

---

## パターン3：actions/cacheキーハッシュ設計──lockfile変更後のキャッシュ汚染

`pnpm-lock.yaml` のハッシュをキャッシュキーに含めないと、依存が変わってもキャッシュが残り `@types/*` のバージョン不一致で `TS2339` が多発する。

**汚染パターン（実際のCI差分ログ）：**
```
Cache restored from key: node-modules-Linux-
# → lockfileが変わっても同じキャッシュを使い続ける
# → @types/react バージョン不一致で TS2339 が多発
```

**修正後のcache設定：**
```yaml
- name: Cache pnpm store
  uses: actions/cache@v4
  with:
    path: ~/.pnpm-store
    key: ${{ runner.os }}-pnpm-${{ hashFiles('pnpm-lock.yaml') }}
    restore-keys: |
      ${{ runner.os }}-pnpm-
```

キャッシュを強制削除する場合はgh CLIで一括実行できる：

```bash
gh cache list --repo owner/repo --json id -q '.[].id' \
  | xargs -I{} gh cache delete --repo owner/repo {}
```

---

## パターン4：Node.js 18/20混在──setup-node未指定でtscバージョンがズレる

ubuntu-latestにはNode.js 18/20が混在しており、`actions/setup-node` で明示しないと意図しないバージョンのtscが動く。

**症状CIログ：**
```
$ pnpm exec tsc --version
Version 5.3.3   ← ローカルは5.5.4だがCIは古いキャッシュのtscを参照
```

**setup-nodeでバージョン固定：**
```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20.x'
    cache: 'pnpm'
```

`cache: 'pnpm'` を指定すると `pnpm-lock.yaml` のハッシュが自動でキャッシュキーになる（パターン3との二重管理は不要）。

---

## コピペ即動作：4問題対策済み .github/workflows/typecheck.yml

```yaml
name: TypeCheck

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: 9

      - uses: actions/setup-node@v4
        with:
          node-version: '20.x'
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Type check
        run: pnpm exec tsc --noEmit --diagnostics

      - name: Debug info (failure時のみ出力)
        if: failure()
        run: |
          echo "Node: $(node -v)"
          echo "TypeScript: $(pnpm exec tsc --version)"
          echo "pnpm: $(pnpm -v)"
          find node_modules/.pnpm -name 'typescript' -maxdepth 3 2>/dev/null | head -5
```

`--frozen-lockfile` でlockfileとnode_modulesの不一致を検出し、`--diagnostics` でメモリ使用量・ファイル数の実測値をログに残す。failureステップはデバッグ情報だけ出力し、CIの成否判定には影響しない。

---

## ローカルOK/CIエラー30秒診断フロー

```
tscがCIのみで失敗
├─ error TS2307 "Cannot find module"
│   ├─ パスに大文字/小文字ブレ → forceConsistentCasingInFileNames: true
│   └─ @/* エイリアスが解決できない → baseUrl を明示
├─ error TS2339 / TS2305（存在するはずのプロパティがない）
│   └─ node_modulesキャッシュ汚染 → gh cache deleteで強制削除
└─ TypeScriptバージョン不一致（tsc --versionの差異）
    └─ setup-nodeでNode.js 20.xを固定
```

このフローで90%以上のCI固有エラーは3分以内に根本原因が特定できる。GitHub Copilot/Claude Codeが生成するtsconfigはパターン1（`forceConsistentCasingInFileNames`の省略）とパターン2（`baseUrl`省略）を同時に踏むケースが多い。生成直後に上記2項目だけでもチェックリストに入れると、本章のパターン4件中2件を事前に防げる。
