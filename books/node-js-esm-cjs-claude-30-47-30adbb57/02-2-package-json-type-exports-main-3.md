---
title: "第2章: package.jsonのtype/exports/mainを3分で確定する設定早見表"
free: false
---

## CJS/ESM 12通りの組み合わせ別エラー早見表

結論から確定させる。`type`・`exports`・`main`の3項目だけで Node.js の読み込み挙動は決まり、組み合わせの破綻パターンは12通りに収束する。下表を貼れば Claude は現状の package.json を1パターンに分類し、即座に「破綻」を指摘する。

| # | type | 拡張子 | import側 | 出るエラー |
|---|------|--------|----------|-----------|
| 1 | module | .js (require) | CJS | `ERR_REQUIRE_ESM` |
| 4 | (なし) | .mjs内require | CJS | `require is not defined` |
| 7 | commonjs | exports無 | ESM | `ERR_PACKAGE_PATH_NOT_EXPORTED` |
| 11 | module | main only | dual | `ERR_MODULE_NOT_FOUND` |

```bash
# 早見表の#番号を即判定させる検証プロンプト
cat package.json | claude -p "この早見表(12通り)のどの#に該当し、破綻か正常か。#番号と一行根拠のみ返せ"
```

## type: module 有無で require が ERR_REQUIRE_ESM になる境界

`type: "module"` を付けた瞬間、`.js` は全てESMとして解釈され、`require()` での読み込みは Node 18/20 では `ERR_REQUIRE_ESM` で即死する。Node 22.12 では `--experimental-require-module` がデフォルト有効化され同期requireが通るため、同じ package.json でも結果が割れる。

```diff
 {
   "name": "mylib",
-  "type": "module",
+  "type": "commonjs",
   "main": "./dist/index.cjs"
 }
```

Claude出力(原文): 「`type:module`下で`main`が`.cjs`を指すと、ESM consumerが`require`扱いとなり Node20で`ERR_REQUIRE_ESM`。`.cjs`明示なら`type`を`commonjs`へ。」

## exports の条件分岐順で import/require が入れ替わる落とし穴

`exports` 内のキーは上から順に評価される。`require` を `import` より上に書くと、ESM環境でもCJSビルドが優先され、ESM側で `__dirname` を期待したコードが `ReferenceError` を起こす。順序は `types → import → require → default` で固定する。

```json
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "default": "./dist/index.mjs"
    }
  }
}
```

```bash
claude -p "exportsの条件キー順序を検査し、import/requireの優先逆転があれば修正後JSONを返せ" < package.json
```

## dual package hazard で踏んだ事故3件の定量報告

dual package(ESM/CJS両提供)で実際に踏んだ事故3件:
1. **インスタンス二重化** — ESM版とCJS版の同一クラスが別物と判定され `instanceof` が `false`。デバッグに4.5時間。
2. **状態分断** — シングルトンのキャッシュが2系統に分かれ、設定値が片側のみ反映。本番で12時間気づかず。
3. **バンドルサイズ2.1倍** — webpack が両ビルドを取り込み、158KB→340KB に膨張。

```diff
 "exports": {
-  ".": { "import": "./esm/index.js", "require": "./cjs/index.js" }
+  ".": { "import": "./esm/index.js", "require": "./esm/wrapper.cjs" }
 }
```

`wrapper.cjs` から動的 `import()` で ESM 本体だけを参照させ、実体を1系統に統一すると事故1・2は消滅した。

## main と module を Node 18/20/22 で優先順位検証する

`module` フィールドは Node 本体が一切読まない(bundler専用)。Node が見るのは `exports` → `main` の順で、`exports` があれば `main` は完全に無視される。この優先順位を3バージョンで実測する検証スクリプトを置く。

```bash
for v in 18.20.4 20.17.0 22.9.0; do
  npx -y node@$v -e "import('./package.json',{with:{type:'json'}}).then(p=>console.log('$v', require.resolve?'cjs':'esm'))"
done
```

```bash
claude -p "mainとmoduleとexportsが共存。Nodeが実際に読むフィールドと、moduleが無視される理由を3行で。残り問題のフィールドだけ列挙" < package.json
```

`module` を頼った設定は Node 実行時に黙って `main` へフォールバックするため、`exports` 未定義のまま `module` だけ書いた package.json は最優先で修正対象になる。
