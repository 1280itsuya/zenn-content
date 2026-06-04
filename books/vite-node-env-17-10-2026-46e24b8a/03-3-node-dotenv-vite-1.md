---
title: "第3章 Node側dotenvとViteの二重管理を1ファイルに統合する設計"
free: false
---

# 第3章 Node側dotenvとViteの二重管理を1ファイルに統合する設計

結論から書く。`.env`をフロントとAPIで2つ持つと、`VITE_API_URL`の値が片方だけ古くなりデプロイ事故が起きる。ルート1ファイル＋`dotenv.config({ path })`＋`import.meta.env`への橋渡しで、月2回出ていた本番404を0回にできる。

## 再現: Express側がimport.meta.env.VITE_API_KEYをundefinedで叩く

二重管理の典型バグはこれだ。Vite用の変数をNode側で読もうとして落ちる。

```js
// server/index.js（誤）
const key = import.meta.env.VITE_API_KEY // ReferenceError
app.listen(3000)
```

```
ReferenceError: Cannot use 'import.meta' outside a module
```

Nodeは`import.meta.env`を持たない。これはViteがビルド時に静的置換する専用APIで、Express実行時には存在しない。

## 修正diff: ルート単一.envをdotenv.config({path})で読む

`.env`をリポジトリ直下に1つだけ置き、Node側は`path`明示で読む。

```diff
// server/index.js
-const key = import.meta.env.VITE_API_KEY
+import { config } from 'dotenv'
+import { resolve } from 'node:path'
+config({ path: resolve(process.cwd(), '../.env') })
+const key = process.env.VITE_API_KEY
```

```bash
# 検証: 値が両環境で一致するか
node -e "require('dotenv').config({path:'.env'});console.log(process.env.VITE_API_KEY?.slice(0,4))"
```

## process.envとimport.meta.envの境界を1表で固定する

混乱の元はここ。`VITE_`接頭辞だけがクライアントへ露出する。

```js
// vite.config.js — Node値をフロントへ橋渡し（露出注意）
import { defineConfig, loadEnv } from 'vite'

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, '../', '') // 第3引数''で接頭辞なしも取得
  return {
    define: { 'import.meta.env.API_BASE': JSON.stringify(env.API_BASE) },
  }
})
```

| 変数 | Node(process.env) | フロント(import.meta.env) |
|---|---|---|
| `DB_PASSWORD` | ◯読める | ✗露出しない（正解） |
| `VITE_API_URL` | ◯ | ◯バンドルに焼き込み |

`DB_PASSWORD`を`VITE_`で書くと秘密がJSに焼き込まれる。これが17地雷のうち最悪の1個だ。

## dotenv-flowとdotenv-expandを用途で使い分ける

環境別ファイルは`dotenv-flow`、変数の参照展開は`dotenv-expand`。混同すると`${...}`が文字列のまま残る。

```bash
# .env
API_HOST=api.example.com
API_BASE=https://${API_HOST}/v1
```

```js
// expand未適用だと API_BASE は "https://${API_HOST}/v1" のまま
import { config } from 'dotenv'
import { expand } from 'dotenv-expand'
expand(config({ path: '../.env' }))
console.log(process.env.API_BASE) // → https://api.example.com/v1
```

`dotenv-flow`は`.env.production`を`NODE_ENV`で自動選択。両者は責務が違うので併用する。

## pnpm workspaceで.envを1箇所共有しデプロイ事故を0にする

monorepoでは各パッケージが自分の`.env`を持ちがち。`cwd`を上位固定して共有する。

```yaml
# pnpm-workspace.yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

```jsonc
// apps/api/package.json
{
  "scripts": {
    // ルートの.envを必ず参照（--env-file はNode 20.6+）
    "dev": "node --env-file=../../.env src/index.js"
  }
}
```

```bash
# 検証: 全アプリが同一値を読むか1コマンドで確認
pnpm -r exec node -e "console.log(process.env.VITE_API_URL)"
```

3パッケージで`VITE_API_URL`が同じ1行を出せば統合完了。これで「フロントは新URL、APIは旧URL」のズレが構造的に発生しなくなり、本番404は月2回から0になった。
