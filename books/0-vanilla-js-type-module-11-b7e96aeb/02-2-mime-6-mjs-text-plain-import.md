---
title: "第2章 MIMEの罠6個｜.mjsがtext/plainで配信されimport全滅する事故"
free: false
---

```markdown
## .mjsがtext/plainで返る瞬間に出る正確なエラー文

結論から言う。`<script type="module" src="app.mjs">` でブラウザが下のエラーを吐くなら、原因はJSの構文ではなくサーバの`Content-Type`だ。MIMEがJavaScript系でなければimport全滅する。

```text
Failed to load module script: Expected a JavaScript module script
but the server responded with a MIME type of "text/plain".
Strict MIME type checking is enforced for module scripts per HTML spec.
```

ES Modulesは通常の`<script>`と違い、MIMEが`text/javascript`等でないと実行を拒否する。DevToolsのNetworkタブで該当`.mjs`を選び、Response Headersの`content-type`を確認するのが切り分けの起点になる。

## 6つの実エラー文と1行修正の対応カタログ

筆者が実環境で踏んだ6件を、再現条件と1行修正で対応させる。

```text
1. MIME "text/plain"         → サーバ既定MIME未設定 → typesに .mjs を追加
2. MIME "application/octet-stream" → 未知拡張子のfallback → 同上
3. MIME "text/html"          → 404をindex.htmlで返却 → ルーティング修正
4. .js なのに module 実行拒否 → 既定が non-JS → .js も text/javascript へ
5. .mjs だけ 404            → 静的配信が .mjs 除外 → mime.types に登録
6. CORS + MIME 二重エラー    → 別オリジン配信 → 同一オリジン化 or CORS付与
```

罠1〜2は配信側のMIME未定義、罠4は`.js`既定の取り違えが原因だ。拡張子を`.mjs`に変えても、サーバが`.mjs`を知らなければ罠2に化けるだけで解決しない。

## Nginxのtypesブロックに1行追記して text/javascript を返す

Nginxは`mime.types`に`.mjs`がない版が多く、これが罠1の最頻原因になる。`http`または`server`ブロックに追記する。

```nginx
# /etc/nginx/conf.d/mjs.conf
types {
    text/javascript js mjs;  # .mjs と .js を両方JSとして配信
}

# 反映: nginx -t && nginx -s reload
```

`curl -sI https://example.com/app.mjs | grep -i content-type` で`text/javascript`が返れば修正完了。5分以内に確定できる。

## Apacheは.htaccessにAddTypeを1行書く

Apache環境(レンタルサーバ含む)は`.htaccess`が使える。`AddType`で拡張子に明示的にMIMEを割り当てる。

```apache
# 公開ディレクトリ直下の .htaccess
AddType text/javascript .js
AddType text/javascript .mjs
```

`AllowOverride None`だと`.htaccess`が無視される点に注意。その場合は管理者がvhost側に同等の`AddType`を入れるしかなく、共有サーバでは罠1が固定化する。

## 4配信環境の.mjsデフォルト挙動を実測比較

同一`.mjs`を4環境にデプロイし、無設定時の`Content-Type`を実測した。

```text
配信環境              | .mjs 既定Content-Type   | type=module
----------------------|-------------------------|------------
GitHub Pages          | text/javascript         | そのまま動作
Cloudflare Pages      | text/javascript         | そのまま動作
Nginx (素のmime.types)| application/octet-stream| 罠2で停止
Apache (AddType無し)  | text/plain              | 罠1で停止
```

`curl -sI` で各環境のヘッダを取って比較した結果だ。GitHub Pages / Cloudflare Pagesは無設定で正しく返すため、ビルド0のVanilla JS配信では検証コストが最小になる。自前のNginx/Apacheへ移す瞬間に罠1・2が再燃するので、移行先のヘッダを`curl`で必ず先に確認しておく。
```
