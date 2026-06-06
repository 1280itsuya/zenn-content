---
title: "第1章 まず動かす：env事故を5分で再現する最小Viteプロジェクトと3つの診断コマンド"
free: true
---

## VITE_接頭辞なしの変数がundefinedになる事故を30行で再現する

結論から言うと、env事故の8割は「Viteが`VITE_`接頭辞のない変数をクライアントバンドルに埋め込まない」仕様を踏んだだけだ。まず手元で再現する。`npm create vite@latest env-trap -- --template vanilla-ts`で雛形を作り、ルートに`.env`を置く。

```bash
# .env
API_URL=https://prod.example.com   # 接頭辞なし → 本番で undefined
VITE_API_URL=https://prod.example.com  # これは埋め込まれる
```

```typescript
// src/main.ts（約10行で事故を観測）
const bad = import.meta.env.API_URL       // undefined になる
const ok  = import.meta.env.VITE_API_URL  // 文字列が入る
document.querySelector('#app')!.innerHTML =
  `<pre>bad=${bad}\nok=${ok}</pre>`
```

`npm run dev`を開くと`bad=undefined`が即座に見える。これが14地雷の0番目だ。

## import.meta.envを丸ごとconsoleに吐く診断スニペット

「どの変数が生き残ったか」を推測で語らない。`import.meta.env`オブジェクト全体をJSONで出力し、埋め込まれたキーを目視する。

```typescript
// src/diagnose.ts
console.table(
  Object.fromEntries(
    Object.entries(import.meta.env).filter(([k]) => k.startsWith('VITE_'))
  )
)
console.log('MODE=', import.meta.env.MODE, 'PROD=', import.meta.env.PROD)
```

`MODE`と`PROD`の値が、devでは`development`/`false`、本番ビルドでは`production`/`true`に変わる。ここが食い違うと2章以降のmode事故に直結する。

## vite build && vite previewでローカルと本番ビルドの値差を可視化する

devサーバーは値を遅延解決するため、事故は本番ビルドで初めて顕在化する。`vite build`した成果物を`vite preview`で配信し、dev時との差分を同じブラウザで比較する。

```bash
npx vite build && npx vite preview --port 4173
# http://localhost:4173 を開き、dev(5173) と表示を見比べる
```

devで出ていた値がpreviewで消えたら、それはバンドル時インライン化の失敗を意味する。

## distをgrepしてenv値がインライン化されたか1コマンドで確認する

最終証拠はビルド成果物そのものにある。Viteは`VITE_`変数を文字列リテラルとして`dist/assets/*.js`に直接書き込む。grepで生存を確認する。

```bash
grep -rо "prod.example.com" dist/assets/ | head
# ヒット0件 = 変数がバンドルに焼き込まれていない（=事故）
```

ヒットすれば値は本番に届く。0件なら接頭辞・mode・読み込み順のいずれかが原因だ。

## --modeでstaging/productionのenvファイルを撃ち分ける

`.env.production`や`.env.staging`を用意しても、`--mode`を渡さなければ読まれない。modeとファイル名の対応を明示してビルドする。

```bash
# .env.staging を読ませる
npx vite build --mode staging
# 読み込み優先順位: .env.staging.local > .env.staging > .env.local > .env
```

この5分の再現環境が、本書の土台になる。次章からは「ビルドは通るのに本番だけ空」「`process.env`が混入してSSRで落ちる」「Dockerのビルド時ARGが効かない」といった実事故14件を、それぞれ再現コードと修正diffで逆引きできる。あなたのプロジェクトで同じ症状が出たら、ここで作った3つの診断コマンドをそのまま当てて、どの地雷を踏んだか1分で特定できる。
