---
title: "第1章 ビルド0で動く最小構成｜index.html+main.mjs+import mapの全文"
free: true
---

## ゴールは「npm install 0回・ビルド0回」でアプリを配信すること

結論を先に置く。本書のゴールは、Viteもwebpackも使わず、`index.html`・`main.mjs`・import mapの3点セットだけでルーティングと状態管理を持つアプリを配信することだ。ビルド工程をゼロにすると、`node_modules`の依存地獄も`dist/`の差分も消える。代わりに`<script type="module">`特有のCORS/MIMEエラーを11個踏むことになる——それを定量カタログ化するのが第2章以降だが、まずは動く最小構成を手元に置く。

以下が配布する`index.html`の全文だ。これ1ファイルで完結する。

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="utf-8" />
  <title>build-zero-app</title>
  <script type="importmap">
  {
    "imports": {
      "preact": "https://esm.sh/preact@10.25.4",
      "preact/hooks": "https://esm.sh/preact@10.25.4/hooks"
    }
  }
  </script>
</head>
<body>
  <div id="app"></div>
  <script type="module" src="./main.mjs"></script>
</body>
</html>
```

## import mapでesm.sh@10.25.4をバンドラ無しで解決する

`type="importmap"`は、`import { render } from "preact"`という裸のモジュール名を、ブラウザがどのURLに解決するかを宣言する仕組みだ。Chrome 89+/Firefox 108+/Safari 16.4+でネイティブ対応済みなので、SystemJSのようなpolyfillは不要。CDNはesm.sh を指定する——unpkgと違い`?dev`や依存の再エクスポートを自動処理するため、`preact/hooks`のようなサブパスも1行で解く。

次が`main.mjs`の全文。状態(`count`)とルーティングの素地をここに持たせる。

```javascript
import { render } from "preact";
import { useState } from "preact/hooks";
import { html } from "https://esm.sh/htm@3.1.1/preact";

function App() {
  const [count, setCount] = useState(0);
  return html`
    <main>
      <h1>build-zero-app</h1>
      <button onClick=${() => setCount(count + 1)}>count: ${count}</button>
    </main>
  `;
}

render(html`<${App} />`, document.getElementById("app"));
```

## python -m http.server 8000 で即起動する

ビルドが無いので、起動はHTTPサーバを1本立てるだけ。Python標準の`http.server`を使えば追加インストールは0だ。`8000`番で配信し、ブラウザで`http://localhost:8000/`を開く。

```bash
# ファイル構成
# .
# ├── index.html
# └── main.mjs

python -m http.server 8000
# → Serving HTTP on :: port 8000 (http://[::]:8000/) ...
```

ボタンを押して`count`が増えれば成功だ。`.mjs`は`http.server`が`Content-Type: text/javascript`を返すため、後述のMIMEエラーを踏まずに済む。

## file:// で開くと CORS で死ぬ理由をエラー文で確認する

「とりあえずダブルクリックで開く」と即死する。`index.html`を`file://`で直接開くと、`type="module"`のフェッチがOriginを`null`扱いし、CORSポリシーで遮断されるからだ。Chromeでは以下が出る。

```text
Access to script at 'file:///C:/build-zero-app/main.mjs' from origin 'null'
has been blocked by CORS policy: Cross origin requests are only supported
for protocol schemes: http, data, isolated-app, chrome-extension, ...
```

`<script type="module">`はクラシックスクリプトと違い、必ずCORSチェックを通る。だから`http://`配信が必須になる。この1点を外すと、以降の検証がすべて再現しなくなる。

## 4つの配信環境のどれを使うかをこの章末で選ぶ

`python -m http.server`はローカル検証用だ。本番は別の選択肢になる。第3章では **(1) Python http.server (2) Node serve@14 (3) GitHub Pages (4) Cloudflare Pages** の4環境で、`.mjs`のMIME・import mapのキャッシュ・SPAフォールバックがどう変わるかを検証済みの設定付きで並べる。たとえばGitHub Pagesは`.mjs`に正しい`text/javascript`を返すが、SPAルーティングは404で落ちる——その回避には次の1行が要る。

```bash
# GitHub Pages の SPA 404 回避: 404.html を index.html のコピーにする
cp index.html 404.html && git add 404.html
```

この最小構成は動く。だが本番配信に載せた瞬間、`Failed to load module script: Expected a JavaScript module script but the server responded with a MIME type of "text/plain"` をはじめ11個のエラーが順に襲ってくる。第2章は、その11個を「再現エラー文 ⇄ 1行修正」のペアで全カタログ化する。手元の3ファイルを開いたまま読み進めてほしい。
