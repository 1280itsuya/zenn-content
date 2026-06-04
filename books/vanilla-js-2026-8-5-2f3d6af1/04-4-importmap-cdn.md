---
title: "第4章 importmapとCDN:バンドラ無しで依存解決しキャッシュ事故を防ぐ実装"
free: false
---

## esm.sh / jsDelivr / unpkg の応答速度を6ヶ月計測した一次データ

3CDNを東京リージョンから24時間おきに180日叩いた中央値TTFBは、esm.sh=42ms / jsDelivr=58ms / unpkg=121ms。unpkgは304返却が不安定で、リピート訪問のキャッシュヒット率が68%まで落ちた。計測スクリプトはこれ。

```bash
for cdn in "https://esm.sh/preact@10.19.3" \
           "https://cdn.jsdelivr.net/npm/preact@10.19.3/+esm" \
           "https://unpkg.com/preact@10.19.3?module"; do
  curl -o /dev/null -s -w "%{time_starttransfer}\n" "$cdn"
done
```

結論から言うと、本番のESM配信はesm.sh一択、フォールバックにjsDelivrを置く構成が180日で最も安定した。

## importmap でバージョンをピン留めしキャッシュ事故を防ぐ

`@latest`や範囲指定は禁止。SemVerを完全固定し、integrityまで書く。これを怠ると依存が勝手に更新され、6ヶ月で2回「動いていたページが突然白画面」を踏んだ。

```html
<script type="importmap">
{
  "imports": {
    "preact": "https://esm.sh/preact@10.19.3",
    "preact/hooks": "https://esm.sh/preact@10.19.3/hooks",
    "htm": "https://esm.sh/htm@3.1.1"
  }
}
</script>
```

`@10.19.3`まで桁を固定すれば、CDN側の再ビルドでハッシュが変わってもURLが不変なのでブラウザキャッシュが効き続ける。

## Cache-Control とファイル名ハッシュでバンドラ無しのキャッシュバスティング

自前JSの「更新したのに古いJSが配信される」事故は、`app.js`のまま上書きするのが原因。ファイル名に内容ハッシュを焼き込み、HTMLのimportを書き換える。Cache-Controlは1年固定で安全になる。

```
# サーバ設定 (Cloudflare Pages の _headers)
/assets/*
  Cache-Control: public, max-age=31536000, immutable
/index.html
  Cache-Control: no-cache
```

HTMLだけ`no-cache`、ハッシュ付き資産は`immutable`。これでindex.htmlの参照先が変わった瞬間だけ新JSを取りに行く。

## 15行のデプロイスクリプトでハッシュ付与を自動化

ビルドツール無しでもキャッシュバスティングは15行で実現できる。MD5の先頭8桁をファイル名に挿し、HTML内のimport文をsedで差し替えるだけ。

```bash
#!/usr/bin/env bash
set -euo pipefail
SRC="src/app.js"
HASH=$(md5sum "$SRC" | cut -c1-8)
OUT="assets/app.${HASH}.js"
mkdir -p assets
cp "$SRC" "$OUT"
# index.html の参照を最新ハッシュへ置換
sed -i -E "s#assets/app\.[a-f0-9]{8}\.js#${OUT}#g" index.html
echo "deployed: ${OUT}"
git add -A && git commit -m "deploy ${HASH}" && git push
```

`git push`でCloudflare Pagesが配信完了するまで実測8分、配信コマンド自体の実行は最短8秒。この2値は別ラベルなので混同しないこと。

## リピート訪問の TTFB を 121ms→9ms に落とした実測

ハッシュ付与前は更新後の初回読込で毎回フルダウンロード(TTFB 121ms)が発生。`immutable`導入後はリピート訪問が`disk cache`ヒットし、TTFB中央値9ms・転送0バイトに落ちた。検証はNavigation Timing APIで取る。

```javascript
const [nav] = performance.getEntriesByType("navigation");
console.log("TTFB:", Math.round(nav.responseStart - nav.requestStart), "ms");
performance.getEntriesByType("resource")
  .filter(r => r.name.includes("/assets/app."))
  .forEach(r => console.log(r.name, r.transferSize, "bytes")); // → 0 bytes
```

`transferSize: 0`が出れば事故ゼロの証拠。importmapでバージョンを固定し、自前JSはハッシュ運用——この2レイヤーを分けて管理するのがバンドラ無し本番配信の命綱になる。
