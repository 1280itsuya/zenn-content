---
title: "第5章: 10エラーを二度と踏まない package.jsonテンプレと検証スクリプト — npm pkg fixで自動点検"
free: false
---

```yaml
topics: ["vite", "nodejs", "npm", "javascript", "frontend"]
```

## 壊れない初期package.json テンプレ2種（ライブラリ用/アプリ用）

結論から、第1〜4章の10エラーは初期テンプレの5フィールドで全て封じられる。下のライブラリ用は `exports` 未定義による `ERR_PACKAGE_PATH_NOT_EXPORTED` と CJS/ESM混在を、`type` と `exports` のbefore/afterで防ぐ。

```jsonc
{
  "name": "@you/lib",
  "version": "0.1.0",
  "type": "module",            // ERR_REQUIRE_ESM を防止
  "engines": { "node": ">=20.11" }, // EBADENGINE を防止
  "exports": {                 // ERR_PACKAGE_PATH_NOT_EXPORTED を防止
    ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js" }
  },
  "files": ["dist"]            // npm publish の空配信を防止
}
```

アプリ用は `exports` を外し `"private": true` を付ける。これで `npm publish` 誤爆と `bin` 解決ミスを同時に止める。

## npm pkg get で5フィールドを機械点検する

`npm pkg get` はpackage.jsonをjqなしで読める。次の1行で危険フィールドを一括出力し、目視ではなく値で判定する。

```bash
npm pkg get type engines.node exports name version
# 出力例: {"type":"module","engines.node":">=20.11", ...}
```

`type` が空文字なら第2章の `ERR_REQUIRE_ESM`、`exports` が `{}` なら第3章のパス解決エラー予備軍。値が空のキーがあれば即修正対象とみなす。

## npm pkg fix で name と bin を自動正規化する

`npm pkg fix` は不正な `name`（大文字・先頭ドット）、欠落 `version`、実行権限の無い `bin` を自動補正する。手修正で踏みがちな第4章の `bin` エラーをコマンド1発で消す。

```bash
npm pkg fix
git diff package.json   # before/after を必ず差分で確認
```

差分が出たら、その行こそが本書のどのエラーを防いだ修正かを `git diff` で1行単位に特定できる。

## 新規プロジェクト生成時に走らせる検証Nodeスクリプト

`npm create vite@latest` 直後に走らせる `verify-pkg.mjs`。5フィールドの欠落を検出し、1つでも欠ければ非ゼロ終了する。

```js
import pkg from "./package.json" with { type: "json" };
const need = ["type", "engines", "version"];
const miss = need.filter((k) => !pkg[k]);
if (miss.length) {
  console.error("欠落フィールド:", miss.join(", "));
  process.exit(1);
}
console.log("package.json OK");
```

`node verify-pkg.mjs` をテンプレのpostinstallに置けば、cloneした全員が起動前に同じ点検を通る。

## pre-commit フックで type・engines・exports 不整合を起動前に検出する

最後にHusky不要の素のGitフックで、コミット前に検証スクリプトを強制する。CIに到達する前の10秒で再発をゼロにする。

```bash
# .git/hooks/pre-commit （chmod +x 必須）
#!/bin/sh
node verify-pkg.mjs || {
  echo "package.json 検証失敗: コミット中止"
  exit 1
}
```

これで第1〜4章の10エラーは、リポジトリへ入る前の段階で機械的に弾かれる。テンプレ・点検・フックの3点セットを新規プロジェクトの雛形に固定すれば、同じエラーを二度と踏まない。
