---
title: "第4章 SPAルーティングとリロード404｜History APIと配信側rewriteの両輪"
free: false
---

## History APIで /about をリロードすると404が返る再現条件

結論: `history.pushState` で `/about` を表示中にブラウザをリロードすると、配信側が `/about` という物理ファイルを探して見つからず404を返す。クライアントルーティング・SPA・リロード404の3点はこの一点に集約される。

最小ルーターはこう書く。ビルドは0、`<script type="module">` から読むだけ。

```javascript
// router.js
const routes = { "/": "ホーム", "/about": "概要", "/posts/1": "記事1" };

function render(path) {
  document.querySelector("#app").textContent = routes[path] ?? "404";
}
document.addEventListener("click", (e) => {
  const a = e.target.closest("a[data-link]");
  if (!a) return;
  e.preventDefault();
  history.pushState(null, "", a.href);
  render(location.pathname);
});
window.addEventListener("popstate", () => render(location.pathname));
render(location.pathname);
```

`pushState` 中は動くが、F5の瞬間にHTTPリクエストが `GET /about` で飛ぶ。ここで配信側の `try_files` フォールバックが無いと事故る。

## Nginx try_files で /about を index.html に1行フォールバック

Nginxは `try_files` で「物理ファイル→ディレクトリ→index.htmlへ」の順に解決させる。これが入っていないと深いURLのリロードが全滅する。

```nginx
server {
  root /var/www/app;
  location / {
    try_files $uri $uri/ /index.html;
  }
  # .js が text/plain で配信されると type=module が落ちるので明示
  location ~ \.js$ {
    add_header Content-Type "text/javascript";
  }
}
```

`$uri` で `/about` の実体を探し、無ければ `/index.html` を返す。これで200が返り、`router.js` の `render(location.pathname)` が `/about` を再描画する。`location ~ \.js$` の `Content-Type` 明示を忘れると第2章のMIMEエラーが再発する。

## Cloudflare Pagesの_redirects と GitHub Pagesの404.htmlハック

配信先が静的ホスティングだと `try_files` は書けない。環境ごとに作法が違う。

Cloudflare Pagesはリポジトリ直下に `_redirects` を1行置く。

```
# _redirects （200指定でリライト＝URLを保持したままindex.htmlを返す）
/*    /index.html   200
```

GitHub Pagesはフォールバック構文が無く、`404.html` をコピーしてJSでパスを復元する。

```bash
# ビルド0なのでcpするだけ。CIなら下記をデプロイ前に実行
cp index.html 404.html
```

GitHub Pagesは404時に `404.html` を200ではなく404ステータスで返す点に注意。SEOを気にする本番では Cloudflare Pages（200リライト）か Nginx を選ぶ。Netlifyは Cloudflare と同じ `_redirects` がそのまま使える。

## import mapのキャッシュで更新が反映されない問題とクエリ版数バスティング

`importmap` で解決したモジュールはブラウザに強くキャッシュされ、`router.js` を直しても古いコードが動き続ける。再現エラーは「コードを直したのに挙動が変わらない」。

```html
<script type="importmap">
{
  "imports": {
    "router": "/router.js?v=20260612"
  }
}
</script>
<script type="module">import "router";</script>
```

`?v=20260612` のクエリ版数を更新するたびにURLが変わり、キャッシュを確実に破棄できる。手動更新は事故るのでデプロイ時に置換する。

```bash
# zenn-cli運用と同じくデプロイ直前にバージョンを刻む
STAMP=$(date +%Y%m%d%H%M)
sed -i "s/router\.js?v=[0-9]*/router.js?v=${STAMP}/" index.html
```

`Cache-Control: no-cache` をHTMLにだけ付け、JSはクエリ版数で破棄するのが、ビルド0で最短のキャッシュバスティングになる。4環境（Nginx / Cloudflare Pages / GitHub Pages / Netlify）すべてでこのクエリ版数方式は等しく機能する。
