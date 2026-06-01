---
title: "第5章 npm publish + GitHub Actions で公開し npx 配布・AI副業テンプレ化する"
free: false
---

```yaml
topics: ["nodejs", "npm", "cli", "githubactions", "ai"]
```

## npm publish 初回設定と files / bin / publishConfig の最小構成

`npm publish` で最初に詰まるのは、ローカルでは動くのに公開後 `npx create-ai-kit` が「command not found」になるパターンだ。原因の9割は `bin` のパス誤りと `files` の配布漏れ。最小の `package.json` は次の5フィールドで決まる。

```json
{
  "name": "create-ai-kit",
  "version": "1.0.0",
  "bin": { "create-ai-kit": "dist/index.js" },
  "files": ["dist"],
  "publishConfig": { "access": "public" }
}
```

`files` に `dist` を入れ忘れると、tarball にソースが含まれず実行不能になる。`npm pack --dry-run` で同梱物を必ず確認する。

## --access public の落とし穴とスコープ付きパッケージの罠

`@your-name/create-ai-kit` のようなスコープ付き名は、デフォルトで private 扱いになり、初回 publish が402エラーで弾かれる。回避策は2つ。

```bash
# 方法1: コマンドで明示
npm publish --access public

# 方法2: package.json に固定（CI向き・推奨）
npm pkg set publishConfig.access=public
```

CI では対話プロンプトが出せないため、`publishConfig` に書く方が事故が少ない。`bin` のファイルには `#!/usr/bin/env node` のシバンを先頭行に置くこと。これが無いと Linux 環境の `npx` で500ミリ秒待たされた末に実行されない。

## GitHub Actions でタグ push 時に自動 publish する CI/CD

`v1.0.0` のようなタグを push した瞬間に publish させる。`NPM_TOKEN` は npm の Automation トークンを発行し、リポジトリの Secrets に登録する。

```yaml
name: publish
on:
  push:
    tags: ["v*"]
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          registry-url: https://registry.npmjs.org
      - run: npm ci && npm run build
      - run: npm publish --provenance
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

`--provenance` を付けると、npm のページに「どの GitHub Actions 実行から出たか」の署名が表示され、配布物の信頼性が上がる。バージョン更新は `npm version patch` で自動コミット＋タグ付けまで一括で済む。

## この雛形を AI 副業ツールの量産起点にする運用設計

`create-ai-kit` を1本作れば、自動執筆ループや投稿 bot は「テンプレ差し替え」で量産できる。`templates/` 配下を切り替える設計にしておく。

```bash
npx create-ai-kit my-writer --template critic-refiner
npx create-ai-kit my-poster --template zenn-deploy
```

第4章で作った CLI 本体は共通のまま、`--template` の中身（GPT-4o-mini judge ループや zenn-cli deploy スクリプト）だけ差す。これで新規ツール1本あたりの初期構築が、毎回30分から3分に短縮される。土台を1度資産化すれば、横展開の限界費用がほぼゼロになる。

## テンプレを収益に変える：有料配布とアフィリ導線の3パターン

公開したテンプレ自体を収入源にする。実測で効くのは次の3導線だ。

```markdown
1. npm は無料公開 → README から有料Bookへ送客（Zenn Book ¥1,000）
2. 拡張テンプレ集を Gumroad で ¥2,980 自動出品
3. README 末尾に開発口座・高還元クレカの計測リンクを設置
```

無料 OSS で月3,000 npx ダウンロードを集め、その1%が ¥1,000 Book を買えば月¥30,000。README の導線には、開発作業の決済に使うネット証券口座や年会費無料・還元率1.5%超のクレジットカード（[A8.net](https://www.a8.net/) 計測リンク）を、ツール紹介の文脈で自然に置く。無料配布で母数を作り、有料テンプレと金融アフィリで回収する二段構えが、起点を継続収入に変える最短経路になる。
