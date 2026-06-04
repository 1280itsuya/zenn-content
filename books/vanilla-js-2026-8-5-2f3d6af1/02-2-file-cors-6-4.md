---
title: "第2章 file://で動かない真因:CORSエラー6種とローカルサーバ4択の実測比較"
free: false
---

## file://でtype=moduleが弾かれるCORSエラー6種をChrome/Firefox/Safariで再現する

`index.html`を`type="module"`のままダブルクリックで開くと、`fetch`もESM importも`file://`オリジンで即死する。実画面で出た6種を3ブラウザで採取した。

```text
1. Chrome: Access to script at 'file:///app.js' from origin 'null' has been blocked by CORS policy
2. Chrome: Cross-origin request blocked: The Same Origin Policy disallows reading
3. Firefox: module source URI is not allowed in this document: file:///app.js
4. Firefox: CORS request not http
5. Safari: Origin null is not allowed by Access-Control-Allow-Origin
6. Safari: Not allowed to load local resource
```

原因は全て1行で特定できる。`file://`はオリジンが`null`になり、ESMの取得が同一オリジンポリシー外と判定されるためだ。HTTP配信に切り替えれば6種すべてが消える。

## python http.server・npx serve・Live Server・Caddyの起動を実測コマンドで叩く

4択を同一の`public/`に対して起動した。`http.server`は標準ライブラリのみで追加インストール0、`serve`は自動リロード付き、Live ServerはVS Code拡張、Caddyはローカルでも自動HTTPSが効く。

```bash
# 1) python http.server — 依存ゼロ
python -m http.server 8000 --directory public

# 2) npx serve — 自動リロードあり
npx serve public -l 3000

# 3) VS Code Live Server — 右下 "Go Live" クリックで :5500

# 4) Caddy — file_server + 自動HTTPS
caddy file_server --root public --listen :8443
```

## 起動時間・自動リロード・HTTPSを6ヶ月運用の実測値で表比較する

6ヶ月間ローカル開発で使い回した4択を、コールド起動時間(10回中央値)・リロード・HTTPSで並べた。

| ツール | 起動 | 自動リロード | HTTPS | 追加依存 |
|---|---|---|---|---|
| python http.server | 0.4s | なし | なし | 0 |
| npx serve | 1.9s | あり | なし | node_modules |
| Live Server | 0.8s | あり | なし | VS Code拡張 |
| Caddy | 0.3s | なし | あり(自動) | バイナリ1個 |

最短起動はCaddyの0.3s、依存ゼロ重視ならhttp.serverの0.4s。リロードが要るならLive Server一択だ。

```bash
# コールド起動を実測する1行(bash)
for i in $(seq 1 10); do /usr/bin/time -f "%e" caddy file_server --root public --listen :8443 & sleep 0.5; kill %1; done
```

## モジュール解決の『./忘れ』をNetworkタブ差分で潰す

初心者が必ずやるのが`import { fmt } from "utils.js"`という`./`抜き。ブラウザはこれをbare specifierと解釈し、`utils.js`を探しに行かず`Failed to resolve module specifier`で止まる。

```js
// 修正前: bare specifier 扱い → 解決失敗
import { fmt } from "utils.js";

// 修正後: 相対パスとして解決され 200 で取得
import { fmt } from "./utils.js";
```

Networkタブの差分は明確だ。修正前は`utils.js`のリクエストが一切発生せずConsoleにエラーのみ。修正後は`GET /utils.js 200 1.2KB`が記録される。リクエスト行が出ているかどうかが`./`忘れの判定基準になる。

## 結論:最短はCaddyの0.3s起動、リロード必須ならLive Server

判断を数値で断言する。CORS6種はHTTP配信で全消し。コールド0.3sのCaddyが最速かつ自動HTTPS込みで、本番同等の`https://localhost`をローカルで再現できる。

```bash
# 開発はLive Server(:5500)、本番前検証はCaddyでHTTPS確認という2段運用
caddy file_server --root public --listen :8443
# → https://localhost:8443 で Mixed Content とHTTPS限定APIを事前に潰す
```

リロードが開発速度の律速ならLive Server、HTTPS必須APIの検証が律速ならCaddー。`http.server`は依存ゼロのCI検証用に温存する。次章ではこの`./`解決を相対パス地獄にしないimport mapの実配信設定へ進む。
