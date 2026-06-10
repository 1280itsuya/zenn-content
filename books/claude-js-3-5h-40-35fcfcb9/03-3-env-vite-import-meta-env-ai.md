---
title: "第3章 .env地獄とViteのimport.meta.env型エラーをAIに直させる"
free: false
---

第3章 .env地獄とViteのimport.meta.env型エラーをAIに直させる

## 結論: .envの事故は「VITE_接頭辞・any型化・本番消失」の3つに集約され、Claudeに型定義とzodバリデーションを生成させれば修正時間は45分→6分に縮む

筆者は12案件で.env関連の手戻りに累計¥18,400を払い、うち¥6,000は本番ビルドで`VITE_API_URL`が`undefined`化した納品後の修正だった。本章はこの3事故をViteの再現コードで潰す。(本章のtopics: `claude / typescript / vite / react / playwright`)

## 事故1: VITE_接頭辞の付け忘れでimport.meta.envがundefinedになる再現コード

Viteは`VITE_`で始まる変数しかクライアントへ露出しない。`.env`に`API_URL=...`と書いて読むと開発も本番も`undefined`になる。

```ts
// src/api.ts ── これは動かない
const base = import.meta.env.API_URL;     // undefined（接頭辞なし）
fetch(`${base}/users`);                    // → "undefined/users" を叩く

// 正しくは .env を VITE_ 付きに直す
// VITE_API_URL=https://api.example.com
const ok = import.meta.env.VITE_API_URL;   // 値が入る
```

## 事故2: import.meta.envがany型化する問題をClaudeにvite-env.d.tsで型付けさせる

デフォルトの`import.meta.env`は`string`索引型で、補完もタイポ検出も効かない。Claudeへ渡すプロンプトはこの1文で済む。

```text
.env の VITE_API_URL と VITE_SUPABASE_KEY を型安全にしたい。
src/vite-env.d.ts に ImportMetaEnv を拡張する型定義を生成して。
```

```ts
// src/vite-env.d.ts ── Claudeが生成
/// <reference types="vite/client" />
interface ImportMetaEnv {
  readonly VITE_API_URL: string;
  readonly VITE_SUPABASE_KEY: string;
}
interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

これで`import.meta.env.VITE_API_RL`（タイポ）が即座に赤線になる。

## 事故3: 本番ビルドで値が消える前にzodで起動時バリデーションする

`vite build`は欠落値を文字列`"undefined"`として埋め込むため、気づくのは納品後だ。zodで起動時に落とせば`¥6,000`の修正は発生しない。

```ts
// src/env.ts
import { z } from "zod";

const schema = z.object({
  VITE_API_URL: z.string().url(),
  VITE_SUPABASE_KEY: z.string().min(20),
});

export const env = schema.parse(import.meta.env);
// 値が欠ければ ZodError で即停止する
```

## .env.exampleとの突合スクリプトで「ローカルにしかない変数」を検出する

納品先で動かない最頻原因は、`.env.example`への書き忘れだ。差分を出すBashを1本置く。

```bash
# scripts/check-env.sh
diff <(grep -oE '^VITE_[A-Z_]+' .env | sort) \
     <(grep -oE '^VITE_[A-Z_]+' .env.example | sort) \
  && echo "OK: 一致" \
  || { echo "NG: 変数の差分あり"; exit 1; }
```

## pre-commitフックをClaudeに生成させて再発を物理的に止める

Playwrightのe2eが通っても.env差分は検知できない。コミット時にcheck-envを強制する。

```bash
# .husky/pre-commit
#!/bin/sh
bash scripts/check-env.sh || exit 1
npx tsc --noEmit            # vite-env.d.ts の型崩れも同時に弾く
```

このフックを入れた13案件目以降、.env起因の手戻りは¥0になった。型定義・zod・突合・フックの4点セットが、副業初動で最も金を溶かす`.env地獄`の出口だ。
