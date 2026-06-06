---
title: "第3章 モードと.env読み込み順の地雷：.env.production.localが効かない4パターンとloadEnv検証法"
free: false
---

## `vite build` が暗黙で `mode=production` になり開発値が混入する4パターン

結論：`vite build` は引数なしでも `--mode production` 相当で走り、`.env.development` の値は読まれない。「ローカルで動いたのに本番で `undefined`」の8割はこれが原因。

```bash
# 開発: mode=development → .env.development を読む
vite
# 本番ビルド: mode=production（暗黙）→ .env.development は読まれない
vite build
# ステージングで開発値を再現したい場合は明示する
vite build --mode development   # ← .env.development.local まで読み込む
```

`import.meta.env.MODE` と `import.meta.env.PROD` をビルド成果物に焼き込んで確認するのが最短。

## `.env.production.local` が効かない優先順位：4ファイルの読込順

Vite の優先度は固定で、後勝ちは「より特化したファイル」。`.env.production.local` > `.env.production` > `.env.local` > `.env`。`.env.local` に古い値が残ると `.env.production` を上書きして「本番に開発値」が出る。

```bash
# 値の決定順を1行で再現（後の echo が勝つ＝優先度高）
for f in .env .env.local .env.production .env.production.local; do
  grep '^VITE_API=' "$f" 2>/dev/null && echo "↑ $f"
done
# VITE_API が複数出たら、最後に出たファイルが実際の採用値
```

## `mode` と `NODE_ENV` は連動しない：`process.env.NODE_ENV` 依存コードの罠

`--mode staging` を渡しても `NODE_ENV` は `production` のまま。`mode` 文字列と `NODE_ENV` は別物で、`if (process.env.NODE_ENV === 'staging')` は永遠に false。

```ts
// ❌ staging を NODE_ENV で判定 → 常に外れる
if (process.env.NODE_ENV === 'staging') enableDebug()
// ✅ Vite が注入する MODE で判定する
if (import.meta.env.MODE === 'staging') enableDebug()
```

## `.gitignore` で `.env.local` がCIに無く値が空になる事故

`.env.local` は Vite 公式テンプレートで gitignore 済み。CIランナーには存在せず `VITE_KEY` が空文字になる。修正diffはCI変数を `.env.production` 経由か Secrets 注入に切り替える。

```diff
- # .env.local（gitignore済・CIに存在しない）
- VITE_API=https://api.example.com
+ # CI では GitHub Actions Secrets を build 前に書き出す
+ # - run: echo "VITE_API=${{ secrets.VITE_API }}" >> .env.production
```

## `loadEnv` で「どのファイルが何順で読まれたか」を実測する

推測をやめ、`vite.config.ts` で `loadEnv` を呼び、採用された変数と空値を起動時にログ出力する。これで地雷を踏む前に検知できる。

```ts
import { defineConfig, loadEnv } from 'vite'

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), 'VITE_')
  console.log(`[env] mode=${mode}`)
  for (const [k, v] of Object.entries(env)) {
    console.log(`  ${k}=${v === '' ? '⚠️空値' : v}`)
  }
  return { define: { __MODE__: JSON.stringify(mode) } }
})
```

`⚠️空値` が出た時点でビルドを止めれば、本番で `undefined` が顧客に届く事故をゼロにできる。`loadEnv(mode, root, '')` と第3引数を空にすれば `VITE_` 接頭辞なしの全変数も検証対象に含められる。
