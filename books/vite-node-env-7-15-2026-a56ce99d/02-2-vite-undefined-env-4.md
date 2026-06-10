---
title: "第2章 VITE_接頭辞漏れでundefined：露出ルールと.env読込4ファイルの優先順位"
free: false
---

以下が第2章の本文です。

---

この章を読み終えると、`import.meta.env.VITE_API_URL` が `undefined` を返す症状を、原因特定からコピペ修正まで90秒で潰せる。さらに4ファイルの読込優先順位を実測ログで確定させ、build後だけ値が消える事故を再現する。

## 症状：開発では効くのにbuildで消えるVITE_接頭辞漏れ

最頻出は接頭辞の付け忘れだ。Viteはクライアントバンドルへ `VITE_` で始まる変数のみ露出する。`API_URL` のままだと `dev` では別経路で拾えるケースがあり、`build` 後に `undefined` 化する。

```bash
# .env
API_URL=https://api.example.com      # ← クライアントから読めない
VITE_API_URL=https://api.example.com # ← これだけが露出する
```

```ts
console.log(import.meta.env.API_URL)      // undefined（接頭辞漏れ）
console.log(import.meta.env.VITE_API_URL) // https://api.example.com
```

判定は1行。`import.meta.env` をまるごと出力し、`VITE_` キーが並ぶか確認する。

```ts
console.table(import.meta.env) // VITE_ で始まるキーだけ表示されればOK
```

## 原因：露出ルールはMODE/PROD/DEV/BASE_URL＋VITE_の5系統だけ

Viteがクライアントへ流すのは組込みの `MODE` `PROD` `DEV` `SSR` `BASE_URL` と、`VITE_` 接頭辞の変数のみ。それ以外は `process.env` 同様サーバ側に留まる。型補完を効かせると漏れに気づける。

```ts
/// <reference types="vite/client" />
interface ImportMetaEnv {
  readonly VITE_API_URL: string
  readonly VITE_GA_ID: string
}
interface ImportMeta { readonly env: ImportMetaEnv }
```

## 修正：envPrefixでAPP_接頭辞へ変える設定

社内規約で `APP_` を使いたい場合は `envPrefix` を上書きする。複数許可も配列で渡せる。

```ts
// vite.config.ts
import { defineConfig } from 'vite'
export default defineConfig({
  envPrefix: ['VITE_', 'APP_'], // APP_API_URL も露出対象になる
})
```

```bash
# .env
APP_API_URL=https://api.example.com # envPrefix変更後はこれが読める
```

## 4ファイルの優先順位：.env.local が .env を必ず上書きする

読込順と上書き挙動を実測する。`npm run build -- --mode production` 実行時、Viteは下記4ファイルを読み、後勝ちで合成する。

```bash
.env                 # 全mode共通・最弱
.env.local           # 全mode共通・gitignore対象
.env.production      # 指定mode限定
.env.production.local # 指定mode限定・最強
```

```ts
// vite.config.ts で実測ログ
import { loadEnv, defineConfig } from 'vite'
export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '')
  console.log('[resolved]', mode, env.VITE_API_URL) // 4ファイル合成後の最終値
  return {}
})
```

同名キーは `.env.production.local` > `.env.production` > `.env.local` > `.env` の順で確定する。

## 事故報告：秘密値にVITE_を付けてバンドルへ漏らし¥48,000請求

`VITE_STRIPE_SECRET` のように秘密鍵へ接頭辞を付けると、minify後の `dist/assets/*.js` に平文で焼き込まれる。あるチームはこれを公開し、不正利用で¥48,000の従量課金を踏んだ。検知は1行で自動化できる。

```bash
# build成果物に秘密値が混入していないか検査（CIへ組込む）
grep -rE "(sk_live|SECRET|PRIVATE_KEY)" dist/ && exit 1 || echo "clean"
```

秘密値は `VITE_` を付けずサーバ専用とし、上記grepをpush前フックに置けば同種事故は再発しない。
