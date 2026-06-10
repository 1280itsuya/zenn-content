---
title: "第1章 まず貼って直す：7症状逆引き.env診断フロー＋vite.config.ts完成形"
free: true
---

## まず結論：あなたのundefinedは「VITE_」接頭辞か参照場所のどちらかで99%直る

`import.meta.env.API_KEY` が `undefined` なら、変数名を `VITE_API_KEY` に変える。`process.env.VITE_xxx` をクライアントコードで読んでいるなら `import.meta.env` に変える。この2つで初期トラブルの大半は5分で消える。残りは下の逆引き表で症状から原因へ飛べる。

## 7症状逆引き：undefined・本番だけ消える・型エラーを原因にマッピング

| # | 症状 | 真の原因 | 即修正 |
|---|------|----------|--------|
| 1 | `import.meta.env.X` が常に undefined | `VITE_` 接頭辞なし | `VITE_X` に改名 |
| 2 | Node側 `process.env.X` だけ undefined | dotenv未読込 | `import 'dotenv/config'` |
| 3 | 本番ビルドだけ値が消える | `.env.production` 不在 | ファイル追加＋再ビルド |
| 4 | TSで `Property does not exist` | 型定義なし | `vite-env.d.ts`（後述） |
| 5 | Dockerで古い値が焼き付く | ビルド時埋め込み | 第2章で詳説 |
| 6 | `.env.local` がCIで効かない | gitignore＋未配置 | CI secretsで注入 |
| 7 | モノレポで読まれない | `envDir` 既定がルート外 | `envDir` 明示 |

症状5・6・7の根因と再発防止は第2章以降で扱う。本章はまず手元の1〜4を潰す。

## クライアントとNodeの境界：import.meta.env と process.env を1図で固定

混乱の正体は「どちらの世界の変数か」を取り違えること。境界はこの1枚で固定できる。

```text
.env  (VITE_API_URL=https://api.example.com / DB_PASS=secret)
        │
        ├─[VITE_ のみ] → ブラウザバンドルに埋め込み → import.meta.env.VITE_API_URL
        │
        └─[全部]       → Nodeプロセスのみ           → process.env.DB_PASS
```

`VITE_` 付きはビルド時にクライアントへ静的展開される＝**ブラウザから丸見え**。`DB_PASS` のような秘密は接頭辞を付けず、Node（APIサーバ）側の `process.env` だけで読む。

## コピペ配布：envPrefix・envDir・loadEnv 設定済み vite.config.ts 全文

```ts
// vite.config.ts
import { defineConfig, loadEnv } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig(({ mode }) => {
  // 第3引数 '' で VITE_ 以外も含め全変数を読む（CI判定などに使う）
  const env = loadEnv(mode, process.cwd(), '')

  return {
    plugins: [react()],
    envDir: process.cwd(),   // モノレポで読めない症状7の対策
    envPrefix: 'VITE_',      // クライアント公開する接頭辞を固定
    define: {
      __APP_ENV__: JSON.stringify(env.APP_ENV ?? 'local'),
    },
  }
})
```

`loadEnv(mode, dir, '')` の第3引数を空文字にすると、`envPrefix` で弾かれる非公開変数も `env` 変数として取得できる。ただし `define` でクライアントへ渡すのは公開してよい値だけに限定する。

## 型エラーを消す：vite-env.d.ts の ImportMetaEnv 定義

症状4（`Property 'VITE_API_URL' does not exist`）は型補完の欠落。このファイルを `src/` 直下に置けば補完が効き、未定義キー参照もコンパイル時に検知できる。

```ts
// src/vite-env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_URL: string
  readonly VITE_GA_ID: string
}

interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

これで `import.meta.env.VITE_API_URL` に型と補完が付き、`VITE_API_ULR` のようなタイポも即座に赤線で出る。

---

ここまでで手元のundefinedと型エラーは消えたはず。だが本当に怖いのは、**ローカルで直ったはずの値がDocker/CIで古いまま焼き付く症状5**だ。`docker build` のレイヤキャッシュと `VITE_` のビルド時埋め込みが噛み合うと、`.env` を変えてもブラウザは旧値を返し続ける——この再現条件と回避手順を、実際に踏んだビルド3回分のログ付きで第2章に記録した。
