---
title: "【2026年最新】Vite+Nodeの.envでハマる全パターン早見表｜import.meta.envとprocess.envが噛み合わない理由を実コードで潰す"
emoji: "🛠"
type: "tech"
topics: ["nodejs","typescript","vite"]
published: true
---
> 本記事は筆者がQiitaで公開した記事のミラーです（原文: https://qiita.com/1280itsuya/items/3a6afc1d24fbcc75dcfc ）。

この記事を読み終えると、「ローカルでは動くのに`npm run build`したら`undefined`」「Nodeのサーバー側で`import.meta.env`が読めない」「`.env`に書いた値が反映されない」という、Vite+Nodeの環境変数あるあるを、自分のプロジェクトで切り分けて直せるようになります。コピペで動く検証コードを2本置いておくので、まず手元で再現させてから読んでください。

## Viteの`import.meta.env`とNodeの`process.env`は別世界だと最初に腹をくくる

結論から断言します。Viteのクライアントコードで`process.env.API_KEY`を書いても、本番ビルドではほぼ確実に`undefined`になります。理由は単純で、`process`はNodeのグローバルであり、ブラウザには存在しないからです。Viteはクライアント向けに`import.meta.env`という別の入口を用意していて、しかも`VITE_`で始まる変数しか露出させません。

ここを混同したまま「`dotenv`を`import`すれば直るはず」と`vite.config.ts`にあれこれ足して泥沼化するのが、初期セットアップで一番多い遭難ルートです。整理すると、登場人物は4つあります。

1. `import.meta.env`（Viteがクライアントに注入。`VITE_`プレフィックス必須）
2. `process.env`（Node実行時のみ。サーバーコード・`vite.config.ts`内）
3. `loadEnv()`（`vite.config.ts`の中で`.env`を手動で読むVite公式API）
4. `dotenv`パッケージ（Node単体スクリプトやExpressサーバーが`.env`を読むため）

この4つの守備範囲が頭に入っていないと、どのファイルにどの書き方をすればいいか永遠に勘で当てることになります。

## VITE_プレフィックスとビルド時インライン化で起きる「ハードコード事故」を再現する

Viteの`import.meta.env.VITE_XXX`は、実行時に解決される変数ではありません。ビルド時に**文字列リテラルとして埋め込まれる（インライン化される）**のがポイントです。つまり`dist/`の中のJSファイルを開くと、`VITE_`付きの値がそのまま生のテキストで入っています。

これは「`VITE_`を付ければクライアントから見える」の裏返しで、**`VITE_`を付けた秘密情報はビルド成果物に丸裸で焼き付く**ことを意味します。`VITE_STRIPE_SECRET_KEY`のような命名をして本番デプロイし、ブラウザのソースに秘密鍵が残っていた、という事故はここから生まれます。

まず挙動を体感するために、最小構成で「焼き付き」を観察します。プロジェクト直下に`.env`を置いてください。

```bash
# .env （プロジェクトルート）
VITE_API_BASE=https://api.example.com
SECRET_TOKEN=this-should-never-reach-the-browser
```

```ts
// src/env-check.ts （Viteクライアント側）
// VITE_ 付きは読める / 付いていない SECRET_TOKEN は undefined になる
const report = {
  apiBase: import.meta.env.VITE_API_BASE,        // => "https://api.example.com"
  secret: import.meta.env.SECRET_TOKEN,          // => undefined（プレフィックス無し）
  legacyProcess: typeof process,                 // => "undefined"（ブラウザにprocessは無い）
  mode: import.meta.env.MODE,                    // => "development" or "production"
  isProd: import.meta.env.PROD,                  // => boolean
};

console.table(report);

// 実証ポイント：build後に検証する
// 1) npm run build
// 2) grep -r "api.example.com" dist/   → ヒットする（インライン化されている証拠）
// 3) grep -r "this-should-never-reach" dist/ → ヒットしない（VITE_無しは注入されない）
export default report;
```

`npm run build`後に`dist/`を`grep`して、`VITE_API_BASE`の値が生で入っていることと、`SECRET_TOKEN`が一切含まれないことを自分の目で確認してください。この一手間を踏むだけで、「秘密は`VITE_`を付けない」「公開してよい設定だけ`VITE_`」という線引きが体に入ります。

## `loadEnv`を使わずに`vite.config.ts`で`process.env`を読もうとすると詰む話

意外と知られていませんが、`vite.config.ts`の中の`process.env`には、`.env`ファイルの値は自動では入っていません。`vite.config.ts`はNodeで評価されますが、Viteが`.env`を読むのは設定評価より後のタイミングであり、しかも読んだ結果を`process.env`に丸投げするわけではないからです。

なので「設定ファイルでポート番号を`.env`から出し分けたい」「`base`や`proxy`先を環境で変えたい」というとき、`process.env.VITE_PORT`を直接参照すると`undefined`で、開発サーバーが意図しないポートで立ち上がります。正解はVite公式の`loadEnv`を使うことです。第3引数を`''`にすると`VITE_`プレフィックス以外も含めて全部読めます。

```ts
// vite.config.ts
import { defineConfig, loadEnv } from 'vite';

export default defineConfig(({ mode }) => {
  // 第3引数 '' で VITE_ 以外（PORT等）も読み込む
  const env = loadEnv(mode, process.cwd(), '');

  // .env に PORT=5180 / VITE_API_BASE=... と書いておけば下が効く
  const port = Number(env.PORT) || 5173;

  return {
    server: {
      port,
      proxy: {
        // サーバー秘密はここで使う：クライアントに焼き付かない
        '/api': {
          target: env.API_ORIGIN || 'http://localhost:3000',
          changeOrigin: true,
        },
      },
    },
    // 必要なら独自プレフィックスを許可（既定は ['VITE_']）
    envPrefix: ['VITE_', 'PUBLIC_'],
  };
});
```

`API_ORIGIN`のようなプレフィックス無しの値は、`proxy.target`（=ビルドに焼き付かないサーバー設定）でだけ使う。これがViteで「秘密をクライアントに漏らさず環境で出し分ける」王道パターンです。`envPrefix`を足せば`PUBLIC_`など自分の命名規則も使えますが、増やすほど露出範囲も広がるので、安易に`envPrefix: ''`で全公開にしないこと。

## `.env.local`と`.env.production`の読み込み優先順位を勘違いして本番URLが上書きされる

Viteは複数の`.env`系ファイルを読みますが、優先順位が直感とズレています。よくある事故は、開発時に作った`.env.local`がGit管理外なのを忘れ、本番ビルド（`mode=production`）でも読まれて、`VITE_API_BASE`がローカルホストのまま本番に出てしまうケースです。

読み込み順（後勝ち）は概ねこうです。

- `.env`（全モード共通・最弱）
- `.env.local`（全モード共通・Git無視が推奨。`.env`より強い）
- `.env.[mode]`（例：`.env.production`。モード限定）
- `.env.[mode].local`（例：`.env.production.local`。最強）

ここで罠なのは、`.env.local`は**`.env.production`より読み込みは先でも、モード固有ファイルである`.env.production`の方が後に評価されて勝つ**点です（同名キーの場合）。つまり「本番値は必ず`.env.production`に書く」「ローカル専用の上書きは`.env.local`」と役割を分ければ事故りません。逆に`.env.local`に本番想定の値を置くと、開発でもCIでも勝手に効いてしまい、原因究明に半日溶かします。`MODE`を`env-check.ts`で出力させ、ビルドのモードを毎回確認する癖を付けてください。

## TypeScriptで`import.meta.env`が`any`になり補完が効かない問題を`vite-env.d.ts`で潰す

TypeScriptプロジェクトだと、`import.meta.env.VITE_API_BASE`が型補完されず`any`扱いになり、タイポしても気づけないという地味な落とし穴があります。`src/vite-env.d.ts`に型を宣言すると、未定義の変数名を書いた瞬間に赤線が出るようになります。

```ts
// src/vite-env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_BASE: string;
  readonly PUBLIC_FEATURE_FLAG?: 'on' | 'off';
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

これで`import.meta.env.VITE_API_BAES`（タイポ）が型エラーになり、`undefined`をビルド成果物に埋め込む前に止められます。`VITE_`変数は注入されないと静かに`undefined`になり実行時まで気づけないため、型で先回りする価値が大きいです。

## Node側（Express等）は`dotenv`が必要、Viteのフロントとは設定を分けて二重管理しない

フロントをVite、APIをExpressのような構成だと、サーバーは`import.meta.env`を持ちません。サーバーは素直に`dotenv`（Node 20.6以降なら`node --env-file=.env`でも可）で`process.env`に読み込みます。

ここでありがちな失敗が、フロントとサーバーで`.env`を2つに分けたのに片方の更新を忘れて、`changeOrigin`先のポートだけ古い値が残る、という不整合です。対策は「公開してよい値は`VITE_`付きで1つの`.env`に集約し、サーバー秘密だけ`.env`の非プレフィックス領域に置く」。`vite.config.ts`は`loadEnv`で同じファイルを読み、サーバーは`dotenv`で同じファイルを読む。読み手は分けても、`.env`という真実のソースは1つにするのがメンテの最短路です。

Node 20.6以降を使っているなら、`dotenv`を入れずにこう起動できます。

```bash
# package.json の scripts 例
# "dev:server": "node --env-file=.env server.js"
# 起動時に .env が process.env へ読み込まれる（dotenv不要）
node --env-file=.env server.js
```

## 5分で切り分けるチートシート（保存版）

手が止まったら上から順に確認してください。

- クライアントで`undefined` → 変数名が`VITE_`始まりか。`.env`を編集したら**devサーバーを再起動**したか（`.env`変更は自動リロードされないことがある）。
- ビルド後だけ壊れる → `process.env`をクライアントで使っていないか。`import.meta.env`へ置換。
- 本番URLがローカルのまま → `.env.local`が勝っていないか。本番値は`.env.production`へ。
- `vite.config.ts`で値が読めない → `process.env`ではなく`loadEnv(mode, process.cwd(), '')`を使う。
- 秘密鍵がブラウザに見える → `VITE_`を外し、`proxy`やサーバー側`process.env`でのみ使用。`dist/`を`grep`して再確認。
- 型補完が効かない → `vite-env.d.ts`に`ImportMetaEnv`を宣言。

Viteの環境変数は「ビルド時インライン化」と「`VITE_`プレフィックスによる露出制御」の2点さえ腹落ちすれば、ほとんどの初期トラブルは10分で切り分けられます。まずは`env-check.ts`を貼って`npm run build`し、`dist/`を`grep`する——この一往復が、勘デバッグから抜け出す一番の近道です。

---

補足：環境構築でローカルマシンやクラウド課金が増えがちな人は、固定費の見直し（格安SIM・光回線）や、開発の合間に始めるネット証券の新NISA口座開設など、生活側の自動化も同時に進めると消耗が減ります。手元のセットアップが片付いたら、そちらの「設定一回で効く系」も棚卸ししてみてください。
