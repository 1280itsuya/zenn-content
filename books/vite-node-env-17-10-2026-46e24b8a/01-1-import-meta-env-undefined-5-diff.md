---
title: "第1章 import.meta.envがundefinedになる5原因と修正diff(無料試し読み)"
free: true
---

## 結論: import.meta.env.VITE_API_KEYがundefinedになるのは99%この5つ

Vite 5系で `import.meta.env.VITE_API_KEY` が `undefined` になる原因は、①`VITE_`接頭辞忘れ ②`vite` 再起動忘れ ③`.env.local` の優先順位誤認 ④型定義欠落 ⑤SSR/CSRの読込差、の5つに収束する。以下すべて「再現→エラーログ→修正diff→検証」の4点セットで30秒ずつ潰す。

## 原因1: VITE_接頭辞なしの環境変数はクライアントに露出しない

`.env` に `API_KEY=xxx` と書いても、Viteはセキュリティ上 `VITE_` 接頭辞のないキーをバンドルへ注入しない。

```bash
# 再現: 値が消える
$ echo "API_KEY=sk-123" > .env
$ node -e "console.log(import.meta)" # ブラウザ側は undefined
```

```diff
- API_KEY=sk-123
+ VITE_API_KEY=sk-123
```

```bash
# 検証
$ grep -r "VITE_" .env && echo "OK"
```

## 原因2: vite起動中の.env編集は再起動まで反映されない

`.env` はHMR対象外。`npm run dev` 起動後に書き換えても、プロセス再起動まで旧値のまま。

```bash
# 検証: 一度落として再起動
$ pkill -f vite ; npm run dev
# 起動ログに "VITE v5.x ready" が出たら反映済み
```

## 原因3: .env.localが.envを上書きする優先順位を誤認する

Viteの読込順は `.env` → `.env.local` → `.env.[mode]` → `.env.[mode].local` で、後勝ち。`.env` を直しても `.env.local` の空値が勝つ。

```diff
# .env.local に古い空値が残っていた
- VITE_API_KEY=
+ VITE_API_KEY=sk-123
```

```bash
$ cat .env.local | grep VITE_API_KEY
```

## 原因4: env.d.tsの型定義欠落でTSが推論を諦めundefined扱いになる

TypeScript環境では型補完がないと `string | undefined` 扱いになり、`as` なしで弾かれる。このテンプレを `src/env.d.ts` に置けば解決。

```typescript
/// <reference types="vite/client" />
interface ImportMetaEnv {
  readonly VITE_API_KEY: string
}
interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

```bash
$ npx tsc --noEmit && echo "型OK"
```

## 原因5: SSR(Node)側はimport.meta.envではなくprocess.envを読む

SSR/SvelteKit/Nuxtのサーバー実行時、`import.meta.env` はビルド時定数で、ランタイム値は `process.env` 側にしか乗らない。

```typescript
// CSR(ブラウザ)
const key = import.meta.env.VITE_API_KEY
// SSR(Nodeランタイム)
const serverKey = process.env.VITE_API_KEY ?? import.meta.env.VITE_API_KEY
```

```bash
$ node -e "console.log(process.env.VITE_API_KEY)" # サーバー側で値確認
```

---

ここまでの5原因は「`dev`環境で見えない」系で、`env.d.ts` テンプレ込みで平均40分の調査が5分に縮む。だが本当に時間を溶かすのは、`npm run dev` では動くのに `vite build` した本番でだけ値が空になるケースだ。第2章では `define` 静的置換の罠・`mode` 不一致・Dockerマルチステージでの `.env` 消失など、本番ビルド限定で値が消える12パターンを同じ4点セットで逆引きする。
