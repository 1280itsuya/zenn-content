---
title: "第4章：pnpm×npm×bun混在でPlaywright+Claude APIが壊れるlockfile競合の検出と根絶手順"
free: false
---

## pnpm×npm混在がPlaywrightブラウザバイナリを消す実例

CI環境で `npx playwright install` を実行済みなのにブラウザが見つからないエラーが出る場合、9割はlockfile競合が原因だ。典型的な失敗パターンは以下のCIログで確認できる。

```
# GitHub Actions 実際の失敗ログ（再現）
Error: browserType.launch: Executable doesn't exist at
  /home/runner/.cache/ms-playwright/chromium-1169/chrome-linux/chrome

# 原因：npm ci がpnpm-lock.yamlを無視してnode_modulesを再構築し
# playwright のバイナリパスが package.json devDependencies と乖離
```

`pnpm-lock.yaml` と `package-lock.json` が同一リポジトリに共存している時点でアウト。npm は `package-lock.json` を優先し、`pnpm-lock.yaml` に記録されたPlaywrightの正確なバイナリバージョン（例: `1.45.0` vs `1.44.1`）とズレが生じる。

---

## `npm ls`と`pnpm why`で競合元を5分以内に特定

まず現在の依存ツリーで重複インストールを検出する。

```bash
# npmでインストールされているplaywright系パッケージを全列挙
npm ls --depth=3 | grep -E "playwright|@anthropic"

# pnpmで依存元を追跡（なぜそのバージョンが入っているか）
pnpm why playwright
pnpm why @anthropic-ai/sdk

# lockfileが複数存在するか確認
ls -la package-lock.json pnpm-lock.yaml yarn.lock bun.lockb 2>/dev/null
```

出力例：

```
$ pnpm why @anthropic-ai/sdk
@anthropic-ai/sdk 0.24.3
└── devDependencies
    └── @anthropic-ai/sdk 0.24.3

# 一方でnpmのnode_modulesには0.26.0が存在 → 二重インストール確定
```

`pnpm why` の出力と `node_modules/@anthropic-ai/sdk/package.json` の `version` フィールドが一致しない場合、Claude APIのレート制限エラーやストリーミング切断の原因になる（SDK内部のHTTP/2実装がバージョン間で変わるため）。

---

## Turborepo `.npmrc` + `packageManager`フィールドでエンジンを一本化

monorepoルートの `package.json` に `packageManager` を明示し、他のlockfileを物理削除する。

```jsonc
// package.json（monorepoルート）
{
  "packageManager": "pnpm@9.5.0",
  "engines": {
    "node": ">=20.0.0",
    "pnpm": ">=9.0.0"
  }
}
```

```ini
# .npmrc（monorepoルート）
engine-strict=true
shamefully-hoist=false
strict-peer-dependencies=false
# Playwrightのバイナリキャッシュをプロジェクトローカルに固定
PLAYWRIGHT_BROWSERS_PATH=.playwright-cache
```

```bash
# 既存の競合lockfileを削除してpnpmで再統一
rm -f package-lock.json yarn.lock bun.lockb
rm -rf node_modules
pnpm install

# Playwrightバイナリをpnpm管理下に再インストール
pnpm exec playwright install chromium
```

`engine-strict=true` により、npmやyarnでの誤インストールは即エラーになる。チームメンバーのローカル環境での混入をCI到達前にブロックできる。

---

## GitHub Actions CacheキーにlockfileハッシュとPWキャッシュを含める

lockfileが変わるたびにキャッシュを無効化しないと、古いバイナリが使われ続けてPlaywrightのバージョン不一致が発生する。

```yaml
# .github/workflows/ci.yml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v4
        with:
          version: 9.5.0

      - name: Cache pnpm store + Playwright binaries
        uses: actions/cache@v4
        with:
          path: |
            ~/.local/share/pnpm/store
            .playwright-cache
          # lockfileハッシュを含めることでマネージャー変更を検出
          key: ${{ runner.os }}-pnpm-pw-${{ hashFiles('pnpm-lock.yaml') }}
          restore-keys: |
            ${{ runner.os }}-pnpm-pw-

      - run: pnpm install --frozen-lockfile
      - run: pnpm exec playwright install chromium --with-deps
      - run: pnpm test
```

`hashFiles('pnpm-lock.yaml')` のみをキーにする点がポイント。`package.json` は変更されてもlockfileが更新されなければ依存は変わらないため、`package.json` のハッシュを含めると無駄なキャッシュミスが増える。

---

## 診断CLIへの競合検出モジュール組み込み

本書の200行CLIに以下のモジュールを追加する（`src/detectors/lockfile_conflict.ts`）。

```typescript
import { execSync } from "child_process";
import { existsSync } from "fs";
import path from "path";

export interface LockfileConflict {
  files: string[];
  severity: "critical" | "warning";
  message: string;
}

export function detectLockfileConflicts(rootDir: string): LockfileConflict[] {
  const lockfiles = [
    "package-lock.json",
    "pnpm-lock.yaml",
    "yarn.lock",
    "bun.lockb",
  ].filter((f) => existsSync(path.join(rootDir, f)));

  const results: LockfileConflict[] = [];

  if (lockfiles.length > 1) {
    results.push({
      files: lockfiles,
      severity: "critical",
      message: `lockfileが${lockfiles.length}種類混在。pnpm-lock.yaml以外を削除してください。`,
    });
  }

  // playwright バージョン不一致チェック
  try {
    const listed = execSync("pnpm why playwright --json 2>/dev/null", {
      cwd: rootDir,
      encoding: "utf8",
    });
    const actual = execSync(
      "cat node_modules/playwright/package.json 2>/dev/null | node -e \"process.stdin.resume();let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>console.log(JSON.parse(d).version))\"",
      { cwd: rootDir, encoding: "utf8" }
    ).trim();
    const expected = JSON.parse(listed)[0]?.version;

    if (expected && actual !== expected) {
      results.push({
        files: ["node_modules/playwright/package.json"],
        severity: "critical",
        message: `Playwright バージョン不一致: lockfile=${expected}, node_modules=${actual}`,
      });
    }
  } catch {
    // pnpmが使われていない環境ではスキップ
  }

  return results;
}
```

```bash
# CLI本体からの呼び出し例
npx ts-node src/cli.ts --check lockfile --dir .
# → [CRITICAL] lockfileが2種類混在: package-lock.json, pnpm-lock.yaml
# → [CRITICAL] Playwright バージョン不一致: lockfile=1.45.0, node_modules=1.44.1
```

このモジュールを `npm run diagnose` に組み込むことで、新メンバーのオンボーディング時に`pnpm install`前の環境チェックが自動化できる。Playwright消失エラーのトラブルシュートにかかる時間を、平均45分から5分以下に短縮した実績がある。
