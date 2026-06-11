---
title: "第1章 5分で潰す『env undefined』Top7とコピペテンプレ"
free: true
---

## 結論: env undefined は7パターンしかない

Vite+Nodeで起動直後に踏む`undefined`は、原因の9割が以下7つに収束する。まず最頻の`VITE_`接頭辞漏れから潰す。Viteはクライアントへ`VITE_`付きしか露出しない。

```diff
# .env
- API_KEY=sk-xxxx          # import.meta.env で undefined
+ VITE_API_KEY=sk-xxxx     # クライアントへ露出する
```

```ts
console.log(import.meta.env.VITE_API_KEY) // "sk-xxxx"
```

## import.meta.env undefined と process.env 取り違え (事故2・3)

`Uncaught ReferenceError: process is not defined`はブラウザ側で`process.env`を読んだ証拠。Vite側は`import.meta.env`、Node(APIサーバ)側は`process.env`と読み分ける。

```diff
- const url = process.env.VITE_API_URL      // ブラウザで爆発
+ const url = import.meta.env.VITE_API_URL  // Vite側はこれ
```

dev起動後に`.env`を編集しても反映されない事故(事故3)は、`vite`の再起動で確定する。HMRは`.env`を監視しない。

```bash
# .env 変更後は必ず再起動
pkill -f vite; npm run dev
```

## .env.local の優先順位と クォート混入 (事故4・5)

`undefined`ではなく`"sk-xxxx"`とクォートごと文字列化される事故が多い。Viteは値を生で読むため、ダブルクォートは値の一部になる。

```diff
# .env.local は .env より優先される(gitignore対象)
- VITE_API_KEY="sk-xxxx"   # 値が `"sk-xxxx"` になる
+ VITE_API_KEY=sk-xxxx
```

```ts
// 取得値が "\"sk-xxxx\"" なら .env のクォートを疑う
console.log(JSON.stringify(import.meta.env.VITE_API_KEY))
```

優先順位は `.env.local` > `.env.[mode]` > `.env`。ローカルだけ古い値を掴むときは`.env.local`の残骸を確認する。

## BOM と CRLF 改行で値が壊れる (事故6・7)

`VITE_API_KEY`の先頭に`\ufeff`が混入し、キー名ごとマッチしなくなる事故。Windowsのメモ帳保存やExcel経由でBOM付きUTF-8になるのが原因。

```bash
# BOMと改行コードを一括除去 (LF・BOMなしへ正規化)
sed -i '1s/^\xEF\xBB\xBF//' .env
dos2unix .env
```

```ts
// 末尾CRが値に残ると "sk-xxxx\r" として比較が落ちる
const key = import.meta.env.VITE_API_KEY?.trim()
```

## コピペ用 最短テンプレ3点セット

7事故を構造的に防ぐ最小構成。`.env.example`で接頭辞を強制し、`env.ts`で型と存在チェックを起動時に効かせる。

```bash
# .env.example (コミットする雛形)
VITE_API_URL=
VITE_API_KEY=
```

```ts
// vite.config.ts — VITE_ 接頭辞だけ露出する既定を明示
import { defineConfig } from 'vite'
export default defineConfig({ envPrefix: 'VITE_' })
```

```ts
// src/env.ts — 起動時に undefined を即死させる型付きアクセサ
const req = (k: string) => {
  const v = import.meta.env[k]
  if (!v) throw new Error(`env missing: ${k}`)
  return v.trim()
}
export const env = { apiUrl: req('VITE_API_URL'), apiKey: req('VITE_API_KEY') }
```

これでローカルは確実に通る。だが`process.env`がCIで空になる、本番ビルドで`import.meta.env`が静的置換され値が固定化する、Dockerの多段ビルドで`.env`が消える——本番とCIで落ちる残り35個は第2章以降で実エラー文付きに逆引きする。
