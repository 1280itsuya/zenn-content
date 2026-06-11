---
title: "第2章 import.meta.env vs process.env 混在事故15例の境界設計"
free: false
---

この章の結論を先に置く。`import.meta.env`はViteがビルド時に文字列置換する公開値、`process.env`はNode実行時に読む秘密値。両者が混ざると「`VITE_`を付けた瞬間に秘密鍵が`dist/assets/*.js`へ静的に焼き込まれる」事故が起きる。判断は1枚のマトリクスで固定し、漏洩は`grep`一発のCIで検知する。

## VITE_接頭辞の有無で秘密鍵がpublic bundleに漏れる境界

Viteは`VITE_`で始まる変数だけを`import.meta.env`経由でクライアントへ露出する。`.env`に置いた`STRIPE_SECRET`を「動かないから」と`VITE_STRIPE_SECRET`へ改名すると、その値は`dist`内のJSへ平文で入る。

```bash
# .env — 名前一文字の差が漏洩の境界
VITE_API_BASE=https://api.example.com   # 公開OK（ブラウザに出る）
STRIPE_SECRET=sk_live_xxxxxxxx          # 秘密（VITE_を付けたら即漏洩）
```

```ts
// クライアント側 src/api.ts
const base = import.meta.env.VITE_API_BASE  // 置換される
const key  = import.meta.env.STRIPE_SECRET  // undefined（接頭辞なしは出ない）
```

## SSRでimport.meta.envがビルド時固定される罠とprocess.envへの退避

SSRサーバが`import.meta.env`を読むと、値は実行時ではなく`vite build`時に固定される。本番のシークレットをローテーションしても旧値が残る原因はこれだ。サーバ側は必ず`process.env`を読む。

```ts
// server/handler.ts — SSRでの正しい読み方
// ❌ const token = import.meta.env.VITE_TOKEN  // build時に焼き込み固定
const token = process.env.API_TOKEN            // ✅ 実行時に評価
```

## loadEnv(mode, process.cwd(), '') で読込範囲を制御する

`vite.config.ts`内では`import.meta.env`は使えない。`loadEnv`の第3引数を`''`にすると接頭辞フィルタを外して全変数を取得でき、`define`で必要な値だけ明示注入できる。

```ts
// vite.config.ts
import { loadEnv, defineConfig } from 'vite'
export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '')  // '' = 全変数。'VITE_'なら公開分のみ
  return {
    define: {
      // server限定値は define せず、流出経路を物理的に断つ
      'import.meta.env.VITE_API_BASE': JSON.stringify(env.VITE_API_BASE),
    },
  }
})
```

## どちらに置くか判断する15事故対応マトリクス

15事故の本質は「公開境界を跨いだか」の1点に収束する。下表を`.env`の先頭コメントに貼り、迷ったら機械的に従う。

```text
値の種類             置き場所     読み方                公開境界
APIベースURL         VITE_xxx     import.meta.env       公開OK
公開アナリティクスID  VITE_xxx     import.meta.env       公開OK
DB接続文字列         接頭辞なし   process.env           秘密
Stripe秘密鍵         接頭辞なし   process.env           秘密
SSRセッション鍵      接頭辞なし   process.env(実行時)   秘密
```

## grep-CIワンライナーで公開バンドルへの秘密混入をゼロにする

ビルド成果物`dist`に秘密パターンが1つでも残ればCIを落とす。`sk_live_`等の実値プレフィックスを直接走査するのが最短で確実だ。

```yaml
# .github/workflows/leak.yml
- name: scan dist for secrets
  run: |
    if grep -rEn 'sk_live_|sk_test_|-----BEGIN|AKIA[0-9A-Z]{16}' dist/; then
      echo "::error::秘密がpublic bundleに混入"; exit 1
    fi
```
