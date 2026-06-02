---
title: "第3章 AIが吐く破綻7選:存在しないAPI・eslintrc旧形式・型不整合の修正秒数"
free: false
---

第3章 本文を以下に示す。

---

結論から言うと、Claude/Copilotの構築コードは「動かない7パターン」に集約される。事前に検出スクリプトを噛ませれば、平均22〜180秒の手戻りをゼロにできる。本章では7破綻すべてに「実エラーログ」「修正秒数」「再発防止プロンプト」を付ける。

## ①.eslintrc旧形式の提案:flat config未対応を23秒で検出

ESLint 9.xはflat config(`eslint.config.js`)必須だが、Claudeは高頻度で`.eslintrc.json`を吐く。

```bash
$ npx eslint .
Oops! Something went wrong! :(
ESLint: 9.21.0
ESLint couldn't find an eslint.config.(js|mjs|cjs) file.
```

検出は1行で済む。

```bash
test -f .eslintrc.json && echo "NG: legacy config (修正23秒)" || echo "OK"
```

再発防止プロンプト:「ESLint 9系のflat config(eslint.config.js)で出力。.eslintrcは禁止」。

## ②存在しないvite pluginのimport:require.resolveで45秒検証

`@vitejs/plugin-react-swc`を`vite-plugin-react-swc`と書き換える幻覚が頻発する。

```ts
// Claudeが吐いた破綻コード
import react from 'vite-plugin-react-swc' // 存在しないパッケージ
```

`npm error 404 Not Found`が出る前に、import名を実在パッケージと突合する。

```bash
node -e "['@vitejs/plugin-react-swc'].forEach(p=>require.resolve(p))" \
  || echo "NG: 幻覚import (修正45秒)"
```

正解は`@vitejs/plugin-react-swc@3.7.0`。プロンプトに「package.jsonに無いパッケージのimport禁止」を明記する。

## ③TS5系で消えた型指定+④peer dependency衝突:90秒の同時退治

TypeScript 5.5以降、`importsNotUsedAsValues`は削除され`verbatimModuleSyntax`へ移行した。Claudeは旧オプションを残す。

```json
{ "compilerOptions": { "importsNotUsedAsValues": "error" } }
```

```bash
$ npx tsc --noEmit
error TS5102: Option 'importsNotUsedAsValues' has been removed.
```

peer衝突(`ERESOLVE`)も同時に潰す検証スクリプト。

```bash
npx tsc --noEmit 2>&1 | grep TS5102 && echo "型破綻 修正60秒"
npm ls --all 2>&1 | grep -i "invalid\|ERESOLVE" && echo "peer衝突 修正30秒"
```

## ⑤Vitest設定の重複と⑥moduleResolution不整合:120秒を1コマンドで

`vite.config.ts`と`vitest.config.ts`の二重定義、`tsconfig`の`moduleResolution: "node"`(本来bundler必須)を一括検出する。

```bash
ls vite.config.ts vitest.config.ts 2>/dev/null | wc -l | grep -q 2 \
  && echo "NG: test設定重複 修正70秒"
grep -q '"moduleResolution": *"node"' tsconfig.json \
  && echo "NG: bundlerにすべき 修正50秒"
```

プロンプトに「Vite 6 + Vitest 3はtest設定をvite.config.tsへ統合、moduleResolutionはbundler」と固定する。

## ⑦package-lockとの乖離を防ぐ統合ゲート

最大の破綻は、生成された`package.json`と`package-lock.json`の不一致(最大180秒)。`npm ci`は乖離で即失敗する。

```yaml
# .github/workflows/ai-guard.yml
- run: |
    npm ci || { echo "lock乖離 修正180秒"; exit 1; }
    bash scripts/ai-7checks.sh  # ①〜⑥を連結
```

この7チェックを案件初動の冒頭に流すだけで、3時間の構築が40分に縮む実測値が再現する。

---

topics: claude, typescript, nodejs, vite, automation
