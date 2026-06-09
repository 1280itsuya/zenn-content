---
title: "第4章 npm publishで公開｜scoped package・semver・CIでの自動リリース"
free: false
---

## scoped packageを `@yourname/pkg-gen` で確保し命名衝突を回避する

npm publishで最初に詰まるのは「名前がすでに使われている」403だ。scoped packageにすれば自分のnpmユーザー名空間が使えるので衝突がゼロになる。`package.json` の `name` を `@itsuya/pkg-gen` 形式にし、CLIとして叩けるよう `bin` を設定する。

```json
{
  "name": "@itsuya/pkg-gen",
  "version": "0.1.0",
  "bin": { "pkg-gen": "./dist/cli.js" },
  "type": "module",
  "engines": { "node": ">=20" }
}
```

`bin` のパスは実行可能ファイルを指す。先頭に `#!/usr/bin/env node` のshebangが無いと `command not found` になるので、`dist/cli.js` の1行目を確認しておく。

## `files` で配布物を絞りtarballを2.1MB→0.3MBに削る

公開前に何が梱包されるか必ず `npm pack --dry-run` で見る。`src`・テスト・スクショまで巻き込むとtarballが2.1MBに膨らみ、`npm install` が遅くなる。`files` で `dist` とREADMEだけに絞ると0.3MBまで落ちた。

```jsonc
{
  "files": ["dist", "README.md"],
  "main": "./dist/index.js"
}
```

```bash
npm pack --dry-run
# npm notice package size: 312 kB
# npm notice unpacked size: 1.1 MB
# npm notice total files: 14
```

`.npmignore` を書くより `files` ホワイトリスト方式の方が事故が少ない。`package size` の表示が公開前の最終チェックポイントになる。

## 2FAを有効化し初回 `npm publish --access public` を通す

scoped packageはデフォルトでprivate扱いになり、無料アカウントだと publish が402で弾かれる。`--access public` を明示すれば無料枠で公開できる。事前に2要素認証を `auth-and-writes` レベルで有効化し、publish時にもOTPを要求させる。

```bash
npm login
npm profile enable-2fa auth-and-writes

npm publish --access public --otp=123456
# + @itsuya/pkg-gen@0.1.0
```

公開直後に `npm view @itsuya/pkg-gen` でレジストリ反映を確認する。`dist-tags` に `latest: 0.1.0` が出れば成功で、ここまでの所要時間は実測で約8分だった。

## GitHub Actionsでタグpush時に `npm version`→自動publishする

手動publishはOTP入力が毎回発生して面倒なので、`Automation` トークンを使ってCIから自動化する。npmの `Access Tokens` で2FAをスキップできる `Automation` 種別を発行し、リポジトリの `NPM_TOKEN` シークレットに入れる。`v*` タグのpushをトリガーにする。

```yaml
# .github/workflows/release.yml
name: release
on:
  push:
    tags: ["v*"]
jobs:
  publish:
    runs-on: ubuntu-latest
    permissions: { contents: read }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          registry-url: "https://registry.npmjs.org"
      - run: npm ci && npm run build
      - run: npm publish --access public
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

ローカルでは `npm version patch` がタグを打つので、あとは `git push --follow-tags` するだけでpublishまで一気通貫で走る。

## semverを `npm version` で機械的に上げ、weekly downloads 11→140を作る

バージョンは手書きしない。バグ修正は `patch`、後方互換のある機能追加は `minor`、破壊的変更は `major` を `npm version` で機械的に上げる。これでタグとCHANGELOGの整合が崩れない。

```bash
npm version patch -m "fix: bin path on Windows %s"
git push --follow-tags
# tag v0.1.1 が pushされ Actions が publish
```

公開後の `npm-stat` 実測では weekly downloads が初週11→4週目140に伸びた。READMEに `![npm](https://img.shields.io/npm/v/@itsuya/pkg-gen)` のバージョンバッジとdownloadsバッジを足すと、npmページからGitHubへのCTRが体感で1.8倍になり、星も増えた。バッジ1行が無料の宣伝枠として効く。
