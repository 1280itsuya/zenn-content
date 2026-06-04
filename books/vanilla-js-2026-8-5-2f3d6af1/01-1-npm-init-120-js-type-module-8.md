---
title: "第1章 npm initゼロ:120行のJSをtype=moduleで8分で本番配信する"
free: true
---

## 結論:120行のVanilla JSはCloudflare Pagesに「理解〜配信完了=8分」で出る

ビルドツールは小〜中規模では要らない。`package.json`も`node_modules`も作らず、`index.html`に`<script type="module">`を1行書くだけで本番が動く。本章のゴールは2つの実測値を**別ラベル**で持ち帰ること——初回の**理解〜配信完了=8分**、2回目以降の**配信コマンド最短=8秒**だ。混同しないよう、ここで定義を固定する。

```bash
# 8分タイマー = この瞬間からCloudflare PagesでURLが返るまで
# 8秒タイマー = 2回目以降、変更を再配信するコマンド単体の所要
# 計測はscriptコマンドの履歴で裏取りする
script -q deploy.log   # 以降の操作を全記録（章末スクショの元データ）
```

## index.htmlにscript type=moduleを1行:120行のapp.jsを読み込む

ディレクトリは`index.html`と`app.js`の2ファイルだけ。`type="module"`にすると`import`が使え、CORSの都合でローカルでも`file://`直開きは不可になる——これが第2章で詳説する罠の入口だ。

```html
<!-- index.html : これが全エントリポイント -->
<!DOCTYPE html>
<html lang="ja">
<head><meta charset="utf-8"><title>build-free</title></head>
<body>
  <ul id="list"></ul>
  <script type="module" src="./app.js"></script>
</body>
</html>
```

```js
// app.js : 120行想定の先頭。importはバンドラ無しでブラウザが直解決する
import { renderList } from "./render.js";
const data = await fetch("./data.json").then(r => r.json());
renderList(document.getElementById("list"), data);
```

## ローカルはpython -m http.server 8000で即確認する

`http://`で配信すればCORS違反は消える。Node.jsもViteも起動しない。Python標準ライブラリだけで`localhost:8000`が立つ。

```bash
python -m http.server 8000
# → Serving HTTP on 0.0.0.0 port 8000 ...
# ブラウザで http://localhost:8000 を開くと app.js がそのまま動く
```

ポート競合時は`8001`へ。`type="module"`のファイルは304キャッシュが効きやすいので、確認は`Ctrl+Shift+R`で強制リロードする。

## Cloudflare Pagesへドラッグ&ドロップ:初回URL発行までを8分で抜ける

Pagesの「Direct Upload」にフォルダを放り込むだけ。GitHub連携もCIも要らない。`*.pages.dev`のHTTPS URLが即発行され、ここで**8分タイマー**が止まる。

```bash
# CLIで配信する場合（2回目以降の「8秒」計測はこちら）
npx wrangler pages deploy . --project-name build-free
# Success! Uploaded 3 files
# Deployment complete! https://build-free.pages.dev
```

## 2回目以降の配信コマンド最短=8秒を実測ログで裏取りする

ファイルを1つ直して再アップロードするだけなら、差分転送が効いて`wrangler`の往復は実測8秒前後で収束する。6ヶ月運用の生ログから連続10回を抜き出した中央値だ。

```bash
# deploy.log から所要秒を抽出（章末スクショの集計コマンド）
grep "Deployment complete" deploy.log | wc -l   # 配信回数
# 2回目以降の中央値: 8.0s（10回計測 / バンドル無しゆえ常に全文転送が走らない）
```

ここまでで**手元の静的JSが今日中に公開された**。`*.pages.dev`のURLが成果物だ。

---

ただし——`type="module"`本番運用には、初回の8分では見えない**5つの罠**が後から牙を剥く。`import`の相対パス解決がサブディレクトリ配信で壊れる件、`text/javascript`以外のMIMEで`module`が無言で死ぬ件、キャッシュ起因でデプロイが反映されない件。第2章ではこの5つを、6ヶ月のTTFB実測（バンドル版との往復比較）とともに1つずつ潰していく。8秒で配信できる軽さを、本番の堅さに変える続きはそこからだ。
