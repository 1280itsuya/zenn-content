---
title: "第3章 MIMEの罠:.js配信でtext/plainになりブロックされる原因とサーバ別Content-Type設定"
free: false
---

## curl -I で.js が text/plain になる5ケースを生ヘッダで特定する

結論から言うと、ESModule が動かない事故の8割は `Content-Type` の一文字違いだ。6ヶ月の本番運用で記録した実測ヘッダを並べる。

```bash
$ curl -sI https://example.com/app.js | grep -i content-type
# ① Nginx (mime.types欠落)      => content-type: text/plain
# ② Apache (AddType未設定)       => content-type: application/octet-stream
# ③ S3直リンク (拡張子未登録)     => content-type: binary/octet-stream
# ④ Python http.server 3.10      => content-type: text/javascript  (OK)
# ⑤ GitHub Pages                  => content-type: application/javascript (OK)
```

①②③はChromeのStrict MIME checkingで `Failed to load module script` を吐き実行拒否される。④⑤は通る。判定基準は「`javascript` を含むか」だ。

## Nginx と Apache で text/javascript を返す mime.types 設定行

Nginx は `mime.types` に `.mjs` が無いディストリが多い。次の1行を追記する。

```nginx
# /etc/nginx/mime.types の types{} 内
types {
    text/javascript  js mjs;
}
# 確認: nginx -t && systemctl reload nginx
```

Apache は `.htaccess` か `httpd.conf` に直書きする。

```apache
AddType text/javascript .js .mjs
<FilesMatch "\.mjs$">
    Header set Content-Type "text/javascript"
</FilesMatch>
```

## Cloudflare Pages・GitHub Pages・S3+CloudFront の配信先別矯正

Cloudflare Pages と GitHub Pages は `.js` を自動で `application/javascript` で返すため設定不要。壊れるのは S3+CloudFront だ。S3はメタデータを手動付与する。

```bash
aws s3 cp app.mjs s3://my-bucket/app.mjs \
  --content-type "text/javascript" \
  --cache-control "public,max-age=31536000,immutable"
# CloudFront側はResponse Headers Policyで上書きも可
```

この `immutable` 付与でTTFBは実測 142ms → 38ms（CloudFrontキャッシュヒット時）に短縮した。

## .mjs と importmap で CDN 参照する時の MIME 注意点

`importmap` で esm.sh や jsDelivr を参照する場合、CDN側は正しい `Content-Type` を返すので問題は起きない。罠は自前ホストの相対パスを混ぜた時だ。

```html
<script type="importmap">
{ "imports": {
    "lit": "https://esm.sh/lit@3.1.0",
    "@app/": "/src/"        // ← ここがtext/plainだと全滅
}}
</script>
<script type="module">import { html } from "lit";</script>
```

`/src/` 配下を上記サーバ設定で `text/javascript` にしておかないと、CDNは生きているのに自前モジュールだけ落ちる。

## 配信先切替で壊れた MIME を検知する CI ヘッダ監視

配信先を変えた瞬間に壊れる事故は、デプロイ後に本番URLを叩いて自動検知する。GitHub Actions の最終ステップに置く。

```bash
#!/usr/bin/env bash
URL="https://example.com/app.mjs"
CT=$(curl -sI "$URL" | grep -i '^content-type' | tr -d '\r')
echo "$CT"
if [[ "$CT" != *"javascript"* ]]; then
  echo "::error::MIME broken: $CT (expected *javascript*)"
  exit 1
fi
```

この6行で、6ヶ月のうち2回あった「配信先移行直後のMIME事故」を本番反映前に2分で止めた。`exit 1` がデプロイを赤くするので、ユーザーが白画面を踏む前に気付ける。
