---
title: "第3章: 『engine node not compatible』『EBADENGINE』— enginesとpackageManager指定でCIだけ落ちる罠"
free: false
---

```yaml
---
topics: ["vite", "nodejs", "npm", "javascript", "frontend"]
---
```

ローカルの `npm run dev` は通るのに、GitHub Actions と Vercel でだけ赤になる。原因の9割は `engines` と `packageManager` の指定ズレだ。この章は3つのエラー文字列を逆引きし、`package.json` の1行差分で消す。

## `EBADENGINE Unsupported engine` はnpm 9以降の警告で止まらない罠

`EBADENGINE` は**エラーではなく警告**で、ローカルの npm install は exit 0 で素通りする。だがCIで `npm ci --engine-strict` を使うと exit 1 に変わる。原因は範囲指定の上限切り忘れだ。

```diff
{
  "engines": {
-   "node": ">=18"
+   "node": ">=18 <23"
  }
}
```

`>=18` だけだと Node 23 の breaking change を許容してしまう。上限を切り、ローカルでも `npm install --engine-strict` で警告を再現させておく。

## `engine "node" is incompatible` でVercelビルドだけ落ちる原因

Vercel は `engines.node` を**メジャー固定**でしか解釈しない。`>=18.17.0` のようなパッチ指定を渡すと `Found invalid Node.js Version` で停止する。

```diff
{
  "engines": {
-   "node": ">=18.17.0"
+   "node": "20.x"
  }
}
```

Vercel が受け付けるのは `18.x` `20.x` `22.x` の3形式のみ。`vercel.json` 側ではなく `package.json` の `engines` が優先される点に注意。

## `Unsupported engine ... packageManager` はCorepack有効化漏れ

`packageManager` フィールドを書いた瞬間、Corepack 未有効の環境では `Cannot find matching keyid` や engine 不整合で落ちる。CIでは明示有効化が必須だ。

```yaml
# .github/workflows/ci.yml
steps:
  - uses: actions/setup-node@v4
    with:
      node-version-file: 'package.json'
  - run: corepack enable
  - run: pnpm install --frozen-lockfile
```

`node-version-file: 'package.json'` で `engines.node` を直読みさせれば、`.nvmrc` との二重管理が消える。

## `.nvmrc` と `engines.node` の数値がズレてCIが赤くなる

`.nvmrc` に `18`、`engines` に `20.x` と書くと、ローカルは18、CIは20で別物がビルドされ、Vite 5 の `crypto.hash` 周りで再現性が崩れる。片方を捨てて1ソース化する。

```bash
# .nvmrc を engines から自動生成し二重管理を撲滅
node -e "console.log(require('./package.json').engines.node.match(/\d+/)[0])" > .nvmrc
cat .nvmrc   # => 20
```

## preinstallスクリプトで`engines`ズレを3秒で検知する

修正後の再発防止は `preinstall` フックが最短だ。`npm install` の度に Node 実バージョンと `engines` を突き合わせ、不一致なら install 前に exit 1 で止める。

```json
{
  "scripts": {
    "preinstall": "node -e \"const s=require('semver'),r=require('./package.json').engines.node;if(!s.satisfies(process.version,r)){console.error('engines violation:',process.version,'!=',r);process.exit(1)}\""
  }
}
```

`semver.satisfies` で範囲評価するため、`20.x` でも `>=18 <23` でも同一ロジックで弾ける。CIの赤を消す最短コマンドは `corepack enable && npm ci --engine-strict` の2連結。これでローカルとCIのNode差分が原因のビルド失敗はゼロになる。
