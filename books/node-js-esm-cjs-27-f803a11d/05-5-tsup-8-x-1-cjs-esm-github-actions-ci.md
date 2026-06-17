---
title: "第5章：tsup 8.xで1コマンドCJS+ESMデュアルビルド→GitHub ActionsでCI自動検証する全設定"
free: false
---

## tsup.config.ts：format:['cjs','esm']と dts の最小構成

tsup 8.xは`format`を配列指定するだけでCJS/ESMを同時出力できる。`dts: true`を添えると型定義（`.d.ts`/`.d.mts`）も両形式で生成され、OpenAIやAnthropicのSDKをラップするライブラリで多発する「型定義がCJS版にしか存在しない」問題を根本から潰せる。

```ts
// tsup.config.ts
import { defineConfig } from 'tsup';

export default defineConfig({
  entry: ['src/index.ts'],
  format: ['cjs', 'esm'],
  dts: true,
  splitting: false,
  sourcemap: true,
  clean: true,
  outDir: 'dist',
  outExtension({ format }) {
    return { js: format === 'cjs' ? '.js' : '.mjs' };
  },
});
```

`outExtension`でCJSを`.js`、ESMを`.mjs`に分けるのが必須。両方を`.js`にすると`package.json`の`exports`フィールドで分岐できなくなる（詰まりポイント①、解決に23分）。

## package.json exportsフィールドを自動生成するNode.jsスクリプト

手書きの`exports`フィールドはエントリポイントが増えるたびに壊れる。ビルド後に走らせるスクリプトで`dist`を走査して`exports`を自動注入する。

```js
// scripts/gen-exports.mjs
import { readFileSync, writeFileSync, readdirSync } from 'fs';

const pkg = JSON.parse(readFileSync('package.json', 'utf8'));
const distFiles = readdirSync('dist');

const exports = {
  '.': {
    import: './dist/index.mjs',
    require: './dist/index.js',
    types: './dist/index.d.ts',
  },
};

for (const file of distFiles) {
  const match = file.match(/^(.+)\.mjs$/);
  if (match && match[1] !== 'index') {
    const name = match[1];
    exports[`./${name}`] = {
      import: `./dist/${name}.mjs`,
      require: `./dist/${name}.js`,
      types:   `./dist/${name}.d.ts`,
    };
  }
}

pkg.exports = exports;
pkg.main   = './dist/index.js';
pkg.module = './dist/index.mjs';
pkg.types  = './dist/index.d.ts';

writeFileSync('package.json', JSON.stringify(pkg, null, 2) + '\n');
console.log('exports generated:', Object.keys(exports).length, 'entries');
```

このスクリプトをCI内でも同一実行することで、手元とCIの`exports`フィールドが常に一致する。

## GitHub Actions matrix: node[18,20,22] × os[ubuntu,windows]でCIを組む

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    strategy:
      matrix:
        node: [18, 20, 22]
        os: [ubuntu-latest, windows-latest]
    runs-on: ${{ matrix.os }}

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'

      - run: npm ci

      - name: Build CJS + ESM
        run: npm run build

      - name: Generate exports
        run: node scripts/gen-exports.mjs

      - name: Verify CJS
        run: node -e "const m = require('./dist/index.js'); console.log('CJS OK:', typeof m)"

      - name: Verify ESM
        run: node --input-type=module -e "import('./dist/index.mjs').then(m => console.log('ESM OK:', typeof m.default))"
```

`windows-latest`を含める理由は、パス区切り文字（`\` vs `/`）の差異でdts生成がWindowsだけ失敗するケースが実在するため（詰まりポイント②、解決に47分）。`node -e`で直接require/importして、壊れたビルドをCIで即検知する。

## セマンティックバージョニングタグトリガーでnpm publishを自動化

```yaml
# .github/workflows/publish.yml
name: Publish

on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write     # npm provenance に必須

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          registry-url: 'https://registry.npmjs.org'

      - run: npm ci
      - run: npm run build
      - run: node scripts/gen-exports.mjs

      - run: npm publish --provenance --access public
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

タグは `git tag v1.0.0 && git push origin v1.0.0` の2コマンドで発火する。`--provenance`フラグはnpm 9.5以降で有効で、npmjs.comのパッケージページにGitHub ActionsのビルドログへのSBOMリンクが表示される。Anthropic SDKが採用しているサプライチェーン攻撃対策で、AI SDKをラップするパッケージには必ず付ける。

## 実際に詰まった3箇所と修正時間の記録

| # | 詰まり箇所 | 症状 | 原因 | 修正時間 |
|---|---|---|---|---|
| ① | `outExtension`省略 | `import('pkg')`がCJS版を読み込む | `.mjs`拡張子なしでexports分岐不能 | 23分 |
| ② | Windows dts生成失敗 | `d.ts`が別ディレクトリに出力される | tsupがWindowsで`/`を`\\`に変換 | 47分 |
| ③ | provenanceでCI失敗 | `ENEEDAUTH: This command requires you to be logged in` | `id-token: write`パーミッション設定漏れ | 11分 |

③はエラーメッセージが認証エラーに見えるため`NPM_TOKEN`のシークレット設定を疑いがちだが、実際には`permissions`ブロックへの`id-token: write`追加だけで解決する。合計81分の実動ログをこの章で全て回収できる構成にした。
