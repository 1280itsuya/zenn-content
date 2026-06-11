---
title: "第3章 dev/preview/build/本番で値が変わる11ハマりとmode設計"
free: false
---

## 第3章 dev/preview/build/本番で値が変わる11ハマりとmode設計

結論を先に出す。`vite dev` で見えていた値が `vite preview`・本番で消える事故の9割は「ビルド時インライン化」と「mode解決順」の2つが原因だ。実行時に変えたい値は `import.meta.env` から外し、`window.__ENV__` で後差しする。この章の判断フローに沿えば、値変更のたびのリビルド3分が0秒になる。

## .env.[mode] の解決順を `--mode staging` で逆引きする

`vite preview` で値が古い時、まず読み込み順を疑う。Viteは `mode` ごとに4ファイルを後勝ちで重ねる。

```bash
# dev は mode=development、build は mode=production が既定
vite build --mode staging   # → .env, .env.staging, .env.local, .env.staging.local
```

```diff
# .env.staging.local が .env.staging を上書きして本番URLが混入していた
- VITE_API_BASE=https://api.example.com   # .env.staging
+ VITE_API_BASE=http://localhost:3000      # .env.staging.local が後勝ち ← 事故
```

`.env.*.local` は最優先かつgit管理外。CIで古い `.local` が残ると本番値が静かに死ぬ。

## `import.meta.env` インライン化で本番のexport差し替えが効かない

`VITE_` 接頭辞の値は**ビルド時に文字列リテラルへ置換**される。だからコンテナ起動時の `export` は届かない。

```ts
// ビルド後の dist には↓がそのまま焼き込まれる
const base = import.meta.env.VITE_API_BASE; // → "https://api.example.com" (固定文字列)
```

```bash
# これは無意味。dist は既にリテラル化済みで上書き不可
docker run -e VITE_API_BASE=https://prod.example.com myapp  # ← 効かない
```

「1イメージを複数環境に流したい」なら、この時点で `VITE_` を諦める判断をする。

## `window.__ENV__` 後差しでリビルド3分→0秒にする

実行時注入したい値は `index.html` にプレースホルダを置き、起動時に置換する。

```html
<!-- index.html -->
<script>window.__ENV__ = { API_BASE: "%API_BASE%" };</script>
```

```bash
# docker-entrypoint.sh: 起動時に環境変数で1ファイルだけ置換
sed -i "s|%API_BASE%|${API_BASE}|g" /usr/share/nginx/html/index.html
exec nginx -g 'daemon off;'
```

```ts
// 参照側。import.meta.env からは外す
const base = (window as any).__ENV__.API_BASE;
```

ビルド成果物は不変のまま環境変数だけ差し替わる。再ビルド3分が `sed` 一発の0秒になる。

## Docker多段ビルドで ARG/ENV が build ステージに届かない

「ローカルでは通るのにCIイメージだけ値がundefined」はステージ間の変数非継承が定番。

```dockerfile
# build ステージの ARG は run ステージへ継承されない
FROM node:20 AS build
ARG VITE_API_BASE          # ← docker build --build-arg が必要
RUN echo "VITE_API_BASE=$VITE_API_BASE" > .env.production && npm run build

FROM nginx:1.27
COPY --from=build /app/dist /usr/share/nginx/html
```

```bash
docker build --build-arg VITE_API_BASE=https://api.example.com -t myapp .
# --build-arg 漏れ → .env.production が空 → 本番で undefined
```

## dev/preview/build の値一致を `vitest` で固定する

「環境ごとに値が変わる」事故は、CIで mode別の解決結果をスナップショット検証して二度と踏まない。

```ts
// loadEnv.test.ts
import { loadEnv } from 'vite';
import { expect, test } from 'vitest';

test('production mode resolves prod API base', () => {
  const env = loadEnv('production', process.cwd(), 'VITE_');
  expect(env.VITE_API_BASE).toBe('https://api.example.com');
});
```

```bash
vitest run loadEnv.test.ts   # CIで mode=production の値ズレを検知
```

判断フローは3分岐で済む。1イメージを複数環境へ→`window.__ENV__`。ビルド時に確定→`VITE_` のまま。秘匿値→そもそもフロントに出さずサーバ側 `process.env` に隔離する。
