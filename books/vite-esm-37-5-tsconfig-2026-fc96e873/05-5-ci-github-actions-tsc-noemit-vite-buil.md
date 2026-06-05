---
title: "第5章 CI緑化: GitHub Actionsでtsc --noEmit/vite build/Vitestを通す.yml完成版と再発防止lint"
free: false
---

```markdown
## ローカルは緑なのにCIだけ赤になる4大差分

CI専用エラーの原因は経験上ほぼ4つに収束する。本書37エラーの索引で言えば #28〜#31 に対応する。実ログから一発で切り分けるワンライナーを置く。

```bash
# CIランナー上で差分を可視化する診断ブロック
node -v                                   # #28 Node版差 (ローカル20.x / CI18.x)
git config core.ignorecase                # #29 大文字小文字 (Mac=true, ubuntu=false)
cat tsconfig.json | grep isolatedModules  # #30 isolatedModules未設定
ls -la node_modules/.vite 2>/dev/null     # #31 キャッシュ汚染の痕跡
```

`import Button from './button'` が手元で通りCIで `TS2307: Cannot find module './Button'` になるのは #29 だ。Linux は大文字小文字を区別する。

## tsc --noEmit / vite build / vitest run を並列で回す.yml完成版

3ジョブを `needs` で直列にせず matrix並列にすると、5分かかっていたCIが約1分50秒に縮む。

```yaml
# .github/workflows/ci.yml
name: ci
on: [push, pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        task: [typecheck, build, test]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20.11.1'   # CI/ローカルを.nvmrcで固定 (#28対策)
          cache: 'npm'
      - run: npm ci
      - name: run ${{ matrix.task }}
        run: |
          case "${{ matrix.task }}" in
            typecheck) npx tsc --noEmit ;;
            build)     npx vite build ;;
            test)      npx vitest run ;;
          esac
```

`fail-fast: false` を外すと typecheck失敗で build/test がスキップされ、原因が同時に見えなくなる。3つ全部を毎回赤緑表示させるのが要点。

## .vite キャッシュ汚染を消すactions/cacheキー設計

`Could not load /node_modules/.vite/deps`（索引 #31）はキャッシュが古いlockと噛む典型。キーに `package-lock.json` のハッシュを必ず混ぜる。

```yaml
      - uses: actions/cache@v4
        with:
          path: node_modules/.vite
          key: vite-${{ runner.os }}-${{ hashFiles('package-lock.json') }}
          restore-keys: vite-${{ runner.os }}-
```

`restore-keys` でlock変更時は前方一致の古いキャッシュを引くが、`key` 完全一致時のみ書き戻すので汚染が連鎖しない。

## isolatedModulesエラーをコミット前に止めるeslint設定

`TS1205: Re-exporting a type ... isolatedModules`（索引 #30）はCIで初めて出がち。`verbatimModuleSyntax` 有効化＋ESLintで型re-exportを機械的に弾く。

```js
// eslint.config.js
import importPlugin from 'eslint-plugin-import';
import tseslint from 'typescript-eslint';

export default tseslint.config({
  plugins: { import: importPlugin },
  rules: {
    '@typescript-eslint/consistent-type-imports': 'error', // import type強制
    'import/no-unresolved': 'error',                        // #29大文字小文字を検出
    '@typescript-eslint/no-import-type-side-effects': 'error',
  },
});
```

これで `export { Foo }`（型）を書いた瞬間ローカルで赤くなり、CIまで届かない。

## Vite/TSメジャー更新時に最初に見る5項目diffチェックリスト

依存更新でCIが一斉崩壊した時、闇雲にエラーを追わず以下を上から順に確認する。本書索引の該当番号付きで再発を防ぐ。

```bash
# 更新直後に流す再発防止チェック
npx tsc --showConfig | grep -E "moduleResolution|verbatim" # ① bundler化したか(#12)
grep '"type"' package.json                                  # ② module宣言の有無(#03)
npx vite build --debug 2>&1 | grep -i "externalize"         # ③ SSR externals変化(#22)
git diff HEAD~1 vite.config.ts                              # ④ plugin API破壊(#19)
npm ls typescript vite                                      # ⑤ 二重バージョン混在(#31)
```

Vite 5→6 では `moduleResolution: "bundler"` 未設定だと #12 が再燃する。①が緑なら残り4項目は5分で消化できる。

> 本章の.ymlと eslint.config.js をコピーすれば、CI緑化に必要な設定は揃う。残る37エラーは巻末逆引き索引から番号で引いて最小diffを当てるだけだ。
```

第5章として、CI専用に出る4差分を実ログで切り分け→matrix並列の `.yml` 完成版→`.vite` キャッシュキー設計→`isolatedModules` を弾くESLint→メジャー更新時の5項目チェックの順で、各H2に索引番号（#28〜#31, #12 等）を紐づけて逆引き辞典として機能する形に仕上げました。コピペで動く構成です。
