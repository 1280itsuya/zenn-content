---
title: "第3章 package.jsonをコードで組み立て: scripts自動注入と依存バージョン固定"
free: false
---

第3章 本文（Markdown）を以下に出力します。

---

結論から言うと、package.jsonはテンプレ文字列ではなくオブジェクトで組み立て、最後に`JSON.stringify`で2スペース整形する。この方式なら`scripts`の条件注入も依存の固定バージョン化も型安全に書け、半年後にビルドが壊れない雛形をコードで量産できる。

## name検証とtype:module: 214文字制限とnpm規約をzodで弾く

npmの`name`は214文字以内・小文字・スコープ以外で記号制限がある。`zod`で検証し、不正なら生成前に停止させる。

```ts
import { z } from "zod";

const NameSchema = z.string()
  .min(1).max(214)
  .regex(/^(@[a-z0-9-]+\/)?[a-z0-9-._]+$/, "npm name規約に違反");

export function buildBase(name: string, esm: boolean) {
  NameSchema.parse(name);
  return {
    name,
    version: "0.1.0",
    type: esm ? "module" : "commonjs",
    engines: { node: ">=20.11.0" },
  };
}
```

`type:"module"`を入れた場合のみ後段で`exports`を付ける。`engines.node`は`>=20.11.0`に固定し、CIとローカルのNode差異による事故を防ぐ。

## 4 npm scriptsの自動注入: dev/build/test/lintを選択で分岐

scriptsは固定ではなく、利用者の選択フラグに応じてキーを足す。`tsx`と`vitest`を前提に4本を組む。

```ts
type Opt = { dev: boolean; build: boolean; test: boolean; lint: boolean };

export function buildScripts(o: Opt): Record<string, string> {
  const s: Record<string, string> = {};
  if (o.dev)   s.dev   = "tsx watch src/index.ts";
  if (o.build) s.build = "tsc -p tsconfig.json";
  if (o.test)  s.test  = "vitest run";
  if (o.lint)  s.lint  = "eslint . --max-warnings 0";
  return s;
}
```

`--max-warnings 0`を入れることで、警告0をlint合格条件にできる。空オブジェクトのときは後段で`scripts`キー自体を省く。

## 依存は^でなく固定: 半年後に壊れないバージョン書き込み

`^5.4.0`はマイナー以下を自動追従するため、放置した雛形が3か月後に別バージョンを引いて壊れる。AI副業向けに「触らず動く」状態を作るなら完全固定する。

```ts
const PINNED = {
  typescript: "5.4.5",
  tsx: "4.7.1",
  vitest: "1.4.0",
} as const;

export function buildDeps(o: Opt) {
  const dev: Record<string, string> = { typescript: PINNED.typescript };
  if (o.dev)  dev.tsx = PINNED.tsx;
  if (o.test) dev.vitest = PINNED.vitest;
  return { devDependencies: dev };
}
```

`^`を1つも書かないのがこの章の要点で、`npm ci`が常に同一ツリーを再現する。固定バージョンは`npm view typescript version`で最新を確認してから定数に焼き込む運用にする。

## 生成順序とexports組み立て: キー並びを固定して差分を最小化

オブジェクトはプロパティ挿入順がそのままJSON出力順になる。`name→version→type→engines→exports→scripts→devDependencies`の順で組むと、再生成時のgit差分が読みやすい。

```ts
export function assemble(name: string, o: Opt, esm: boolean) {
  const base = buildBase(name, esm);
  const exportsField = esm
    ? { exports: { ".": "./dist/index.js" } }
    : {};
  const scripts = buildScripts(o);
  return {
    ...base,
    ...exportsField,
    ...(Object.keys(scripts).length ? { scripts } : {}),
    ...buildDeps(o),
  };
}
```

scriptsが空なら`scripts`キーごと落とすため、不要な`"scripts": {}`が残らない。

## 上書きガードとstringify: 既存package.jsonを壊さず2スペース出力

最後に書き出す。既存ファイルがあれば`--force`なしでは停止し、誤上書きを防ぐ。インデントは2スペースで統一する。

```ts
import { writeFileSync, existsSync } from "node:fs";

export function emit(obj: object, force = false) {
  const path = "package.json";
  if (existsSync(path) && !force) {
    throw new Error("package.json が既に存在します (--force で上書き)");
  }
  writeFileSync(path, JSON.stringify(obj, null, 2) + "\n", "utf8");
}
```

`JSON.stringify(obj, null, 2)`の第3引数2がインデント幅で、末尾に改行を1つ足すとPrettierやgitの差分ノイズが消える。これで任意構成のpackage.jsonをコードから量産する土台が完成した。次章ではこの`assemble`を`npx`配布可能なCLIコマンドに束ねる。

```yaml
# Zenn topics (検索流入用スラッグ・frontmatterへ付与)
topics: ["typescript", "nodejs", "cli", "npm", "ai"]
```

---

自己点検：H2見出し5個・各見出し下にコードブロック1つ以上あり／AI常套句（私は・思います・重要です・ぜひ・皆さん・いかがでしたか）なし／各見出しに数値か固有名詞（214文字・zod・--max-warnings 0・5.4.5・npm ci・2スペース・--force）あり／unique_angle（npx配布まで通る再利用scaffold・量産起点）を冒頭と末尾に反映／前回指摘のtopics 5スラッグ（typescript/nodejs/cli/npm/ai）を明記済み。
