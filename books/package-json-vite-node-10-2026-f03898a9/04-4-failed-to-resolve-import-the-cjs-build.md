---
title: "第4章: 『Failed to resolve import』『The CJS build of Vite deprecated』— depsとscriptsの落とし穴4選"
free: false
---

```yaml
topics: ["vite", "nodejs", "npm", "javascript", "frontend"]
```

## `Failed to resolve import` はdependencies誤配置が9割

結論から言うと、このエラーの大半は実行時に必要なパッケージを`devDependencies`へ入れたことが原因だ。Viteのpre-bundle(esbuild)は`dependencies`を基準に依存グラフを辿るため、`react`や`axios`がdev側にあると本番ビルドで解決に失敗する。

```diff
 // package.json
   "dependencies": {
+    "axios": "^1.7.9"
   },
   "devDependencies": {
-    "axios": "^1.7.9",
     "vite": "^6.0.7"
   }
```

移動後に`rm -rf node_modules/.vite && npm i`でpre-bundleキャッシュを破棄すれば復旧する。

## `The CJS build of Vite's Node API is deprecated` の消し方

Vite 5以降で出るこの警告は、設定ファイルがCJSとして読まれている合図だ。放置するとVite 7で完全に動かなくなる。`type: "module"`の1行を足すのが最短修正となる。

```diff
 // package.json
 {
   "name": "my-app",
+  "type": "module",
   "version": "0.0.1"
 }
```

`vite.config.js`を`.mjs`に拡張子変更しても消えるが、ESM化に伴い`require`が使えなくなるため、設定内の`require('path')`は`import path from 'node:path'`へ置換する。

## peerDependencies競合は`overrides`で1行解決

`npm i`時の`ERESOLVE unable to resolve dependency tree`は、プラグインが要求するViteのバージョン範囲が食い違うと発生する。`--force`は破綻の先送りなので、`overrides`で明示的にバージョンを固定する。

```diff
 // package.json
+  "overrides": {
+    "vite": "$vite"
+  },
   "devDependencies": {
     "vite": "^6.0.7",
     "vite-plugin-pwa": "^0.21.1"
   }
```

`"$vite"`は自プロジェクトの`vite`を全依存へ強制する記法。pnpmなら`pnpm.overrides`、yarnなら`resolutions`へ同義の指定を移す。

## `npm run dev`が無反応ならscriptsのバイナリ解決を疑う

ターミナルが沈黙する症状は、グローバルの`vite`を見に行って失敗しているケースが多い。`scripts`は`node_modules/.bin`を優先するため、コマンドは`vite`単体で書き、絶対パスを書かない。

```diff
 // package.json
   "scripts": {
-    "dev": "npx vite --port 3000",
+    "dev": "vite --port 3000",
     "build": "vite build"
   }
```

ロックファイル不整合(`package-lock.json`と`pnpm-lock.yaml`が両在)も無反応の典型。下記チェックで削除可否を判断する。

```bash
# どのロックを残すか1コマンドで確定
ls package-lock.json pnpm-lock.yaml yarn.lock 2>/dev/null
# → 複数出たら使うPM以外を削除し、node_modules ごと再生成
rm -rf node_modules package-lock.json && pnpm install
```

複数のロックファイルが残ると`.bin`内のシンボリックリンクが壊れ、`run dev`がエラーも吐かず即終了する。使うパッケージマネージャを1つに固定し、対応するロックだけをコミット対象に残すのが再発防止の決め手だ。
