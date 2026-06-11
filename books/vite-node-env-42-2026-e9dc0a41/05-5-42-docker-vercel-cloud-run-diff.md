---
title: "第5章 42事故の逆引き表とDocker/Vercel/Cloud Run別デプロイdiff"
free: false
---

第5章の本文です。

---

この章は、前4章で潰した42事故を「エラー文で検索→該当diff行に飛ぶ」逆引き索引に圧縮する。さらにVercel/Cloud Run/Docker Composeの3環境別に、`import.meta.env`と`process.env`が混在するViteプロジェクト固有のデプロイ罠を最小diffで示す。コピペで動く.envテンプレ群はDLリンクで配布する。

## 42事故をエラー文で引く逆引き索引(grep一発・章ジャンプ)

事故番号・症状・修正章・diff行を1枚のTSVにした。エラー全文をうろ覚えでも、特徴語をgrepすれば30秒で着地できる。

```bash
# index.tsv: id<TAB>error_substr<TAB>chapter<TAB>diff_line
grep -i "import.meta.env" index.tsv
# => 07  import.meta.env undefined  ch2  L48: define->envPrefix
grep -i "process is not defined" index.tsv
# => 19  process is not defined  ch3  L112: import.meta.env移行
```

`VITE_`接頭辞欠落(事故07)とブラウザでの`process`参照(事故19)がViteで最頻の2大事故。error_substr列は実スタックトレース文字列をそのまま入れてあるので、貼り付けて`grep -F`でも当たる。

## Vercel: VITE_接頭辞・環境スコープ・vercel envの罠

Vercelは`VITE_`の付かない変数をビルド時にクライアントへ注入しない。Production/Preview/Developmentのスコープ取り違え(事故31)も多発する。

```bash
vercel env add VITE_API_BASE production
vercel env ls   # build時に値が空ならスコープ違い
```

```diff
# vite.config.ts — Vercelで env が空になる時の確定diff
- envPrefix: 'APP_',
+ envPrefix: ['VITE_', 'APP_'], // 既存APP_を残しつつVITE_を解禁
```

固有罠: `vercel dev`はローカル`.env`を読むが、本番は管理画面の値が優先。同名キーがあると静かに本番値で上書きされる。

## Cloud Run: ビルド時注入とruntime envの分離(gcloud builds)

Cloud Runの`--set-env-vars`はコンテナ起動時(runtime)にしか効かない。Viteは`vite build`時に値を焼き込むため、runtime envはバンドルに入らない(事故38)。

```bash
# NG: VITE_*がフロントに反映されない
gcloud run deploy app --set-env-vars VITE_API_BASE=https://api.example.com
# OK: ビルド引数で渡しDockerfileのbuild段で展開
gcloud builds submit --substitutions=_VITE_API_BASE=https://api.example.com
```

```dockerfile
ARG _VITE_API_BASE
ENV VITE_API_BASE=$_VITE_API_BASE
RUN npm run build   # ここで初めて値が焼き込まれる
```

## Docker Compose: env_fileとbuild.argsの二重ミス

Composeの`env_file`はruntime用、`build.args`はビルド用。Viteフロントは後者でないと届かない(事故40)。Compose直下`.env`との混同(事故41)も定番。

```yaml
services:
  web:
    build:
      context: .
      args:
        VITE_API_BASE: ${VITE_API_BASE}  # ホスト.envから注入
    env_file: .env.runtime               # Node側process.env用
```

固有罠: Compose直下の`.env`は変数展開専用でコンテナに入らない。フロントに渡すなら必ず`build.args`へ橋渡しする。

## 0から落とさない18項目チェックリストと.envテンプレ配布

新規プロジェクトを事故ゼロで立ち上げる検証スクリプト。CIに置けば接頭辞漏れを起動前に弾ける。

```bash
#!/usr/bin/env bash
# preflight.sh — 18項目のうち最頻3つを自動検査
set -e
grep -rq "process.env" src/ && echo "WARN: src内process.env(事故19候補)"
test -f .env.example || { echo "NG: .env.example欠落(事故12)"; exit 1; }
grep -q "^VITE_" .env || echo "WARN: VITE_接頭辞ゼロ(事故07候補)"
echo "preflight done"
```

残り15項目の完全版と、3環境別`.env.example`/`.env.runtime`テンプレ一式は下記から取得できる。コピー後に`preflight.sh`を通せば、42事故中セットアップ起因の27件は着手前に消える。

```bash
# テンプレ一式DL(本書購入者向け)
curl -L https://example.com/vite-env-kit.zip -o vite-env-kit.zip
unzip vite-env-kit.zip   # .env.example / preflight.sh / compose.yaml 同梱
```
