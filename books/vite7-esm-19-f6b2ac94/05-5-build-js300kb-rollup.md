---
title: "第5章 本番buildでJS300KB削るrollup設定と計測の数字"
free: false
---

## まず計測しないと削れない: rollup-plugin-visualizer で 1.2MB の内訳を出す

`vite build` が想定の2倍に膨れる原因の8割は vendor の二重バンドルだ。まず可視化する。

```bash
npm i -D rollup-plugin-visualizer@5.12
```

```ts
// vite.config.ts
import { visualizer } from "rollup-plugin-visualizer";
export default defineConfig({
  plugins: [
    visualizer({ filename: "dist/stats.html", gzipSize: true, brotliSize: true }),
  ],
});
```

`vite build` 後に `dist/stats.html` を開くと、`lodash` 全体 (71KB gzip) や `moment` (低レート時 68KB) が丸ごと入っているのが面積で分かる。計測前のサイズは `dist/assets/index-*.js` が **gzip 412KB / raw 1.21MB** だった。

## manualChunks で vendor を切り出し JS を 300KB 削る

単一チャンクだと1行変えるたびに全 JS のキャッシュが飛ぶ。`manualChunks` で安定ライブラリを分離する。

```ts
build: {
  rollupOptions: {
    output: {
      manualChunks(id) {
        if (id.includes("node_modules")) {
          if (id.includes("react")) return "react-vendor";
          if (id.includes("chart.js")) return "chart-vendor";
          return "vendor";
        }
      },
    },
  },
},
```

分割後、初期ロードに必要な `index` チャンクは **gzip 412KB → 118KB**、`react-vendor` (142KB) は再訪時キャッシュヒットで再ダウンロードなし。`moment` を `dayjs` (2.8KB) に置換して raw で **約300KB** 削れた。

## build.target を es2020 に上げて polyfill 33KB を捨てる

target が低いと Vite が不要な `regenerator-runtime` などを注入する。`Chrome 87+` だけ見るなら ESM ネイティブで足りる。

```ts
build: {
  target: "es2020",        // デフォルトの "modules" より明示が安全
  cssTarget: "chrome87",
  modulePreload: { polyfill: false }, // polyfill 約33KB を除去
},
```

`Top-level await is not available in the configured target environment` が出たら target が低すぎる証拠なので `es2022` に上げる。これで legacy polyfill 33KB が消える。

## dynamic import でルート単位に分割し初期 JS を 118KB→64KB

全画面を起動時に読む必要はない。`import()` でルートを遅延ロードする。

```ts
const Dashboard = lazy(() => import("./pages/Dashboard")); // 別チャンクに自動分割
const Report = lazy(() => import("./pages/Report"));
```

`vite build` のログで `Dashboard-*.js 54KB` が独立チャンクになったことを確認。トップ画面の初期 JS は **118KB → 64KB**、Lighthouse の TBT が **310ms → 90ms** に落ちた。

## source map を本番配信から外し 1.8MB の転送を止める

`build.sourcemap: true` のまま deploy すると `.map` が公開され、配信サイズに **約1.8MB** 上乗せされる。

```ts
build: {
  sourcemap: process.env.NODE_ENV === "production" ? "hidden" : true,
},
```

`"hidden"` は map を生成しつつ JS 末尾の `//# sourceMappingURL=` を消すため、Sentry へアップロードしてからエラー追跡だけ残せる。`dist` から `.map` を外部公開しない設定だ。

---

ここまでの計測手順を自分のアプリで回し続けるには、Rollup/Vite の内部を体系的に押さえておくと逆引きが速くなる。実行計画やバンドル解析の学習には技術書 ([Amazon 開発書籍](https://www.amazon.co.jp/)) が効率的で、計測〜CI 自動化までまとめて組みたいなら AI 自動化教材 ([Gumroad ai_kit](https://gumroad.com/)) のテンプレートが build 計測スクリプトの雛形として使える。
