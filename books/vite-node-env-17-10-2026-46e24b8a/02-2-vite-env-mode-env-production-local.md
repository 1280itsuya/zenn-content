---
title: "第2章 Viteの.env優先順位とmode別読込(.env.production/.local)を完全図解"
free: false
---

## 結論: Viteは4ファイルを「下→上」で上書きし、`.local`が常に勝つ

Viteの`.env`読込は以下の優先順位（数字が大きいほど強い）。`--mode`未指定なら`mode=development`。

```bash
# 検証コード: 同じキーVITE_APIを4ファイルに別値で書く
echo "VITE_API=base"            > .env
echo "VITE_API=local"          > .env.local
echo "VITE_API=prod"           > .env.production
echo "VITE_API=prod-local"     > .env.production.local
```

| 順位 | ファイル | `vite` (dev) | `vite build` (production) |
|---|---|---|---|
| 1 | `.env` | base | base |
| 2 | `.env.local` | **local** | prod-local が勝つので無視 |
| 3 | `.env.[mode]` | 読まれない | prod |
| 4 | `.env.[mode].local` | 読まれない | **prod-local** |

dev では `local`、build では `prod-local` が最終値になる。

## `--mode staging`を渡すと`.env.staging`が`.env.production`より勝つ

`vite build`のデフォルトmodeは`production`だが、`--mode`で差し替えられる。

```bash
# .env.staging を用意して staging ビルド
echo "VITE_API=staging" > .env.staging
npx vite build --mode staging
```

このとき読まれるのは `.env` → `.env.local` → `.env.staging` → `.env.staging.local`。`.env.production`は**一切読まれない**。エラーログの典型は本番URLが混入する事故：

```text
[検証] dist/assets/index-xxxx.js に "prod" が残存
→ grep "VITE_API" dist/assets/*.js が staging ではなく prod を出力
```

```bash
# 検証コマンド: ビルド成果物に正しい値が焼かれたか確認
grep -ro "VITE_API[^,]*" dist/assets/ | grep -v staging
# 出力が空ならOK。1行でも出たら mode 指定漏れ
```

## `vite.config.ts`では`import.meta.env`がundefinedになる→`loadEnv()`を使う

設定ファイル内で`import.meta.env.VITE_API`を読むと`undefined`。configはNode側で評価されるため。

```diff
- // vite.config.ts — これは動かない
- export default defineConfig({
-   server: { port: import.meta.env.VITE_PORT }  // undefined
- })
+ import { defineConfig, loadEnv } from 'vite'
+ export default defineConfig(({ mode }) => {
+   const env = loadEnv(mode, process.cwd(), '')  // 第3引数''で全prefix読込
+   return { server: { port: Number(env.VITE_PORT) || 5173 } }
+ })
```

```bash
# 検証: configが正しい値を拾うか
npx vite build --mode staging --debug 2>&1 | grep "port"
```

`loadEnv`の第3引数を`''`にしないと`VITE_`以外を読めない点が事故ポイント。

## `.env.local`のgitignore漏れで本番キーをpush→¥48,000請求された再現

`.env.local`は機密用だが、`.gitignore`に1行追加し忘れると公開リポジトリに本番APIキーが乗る。

```bash
# 事故の再現: ignore漏れ状態でコミットされたか監査
git ls-files | grep -E '\.env(\.local|\.production\.local)?$'
# ここに .env.local が出たら漏洩済み
```

実例ではOpenAIキーが3日間公開され、クローラに拾われて`¥48,000`の不正利用が発生。修正diff：

```diff
# .gitignore
+ *.local
+ .env.*.local
```

```bash
# 既にpush済みなら履歴から除去 + キー即時revoke
git rm --cached .env.local && git commit -m "remove leaked env"
```

## `envPrefix`を`APP_`に変えると踏む3つの落とし穴

`VITE_`接頭辞を嫌って`envPrefix`を変更できるが、3点で詰まる。

```ts
// vite.config.ts
export default defineConfig({ envPrefix: 'APP_' })
```

```diff
# 落とし穴1: 既存の VITE_ 変数が全て露出しなくなる
- import.meta.env.VITE_API  // undefined
+ import.meta.env.APP_API

# 落とし穴2: 'ENV_' など Vite予約に近い名前は警告
- envPrefix: 'ENV_'  // Error: blocked, conflicts with internal
+ envPrefix: 'APP_'

# 落とし穴3: 空文字 '' を渡すと全env露出=機密漏洩
- envPrefix: ''  // PATH や SECRET まで dist に焼かれる
+ envPrefix: 'APP_'
```

```bash
# 検証: 意図しない変数が露出していないか
npx vite build && grep -o "SECRET\|PATH=" dist/assets/*.js
# 出力が空なら prefix 設定は安全
```

`envPrefix`変更時は必ずこの`grep`で漏洩監査を回すこと。空文字指定は1回で全秘匿値を焼くため`config`レビューの必須チェック項目にする。
