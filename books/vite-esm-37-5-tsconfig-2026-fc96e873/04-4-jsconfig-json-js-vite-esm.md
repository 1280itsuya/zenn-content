---
title: "第4章 jsconfig.json派の罠: 型なしJSプロジェクトでViteのESM補完を効かせる最小設定"
free: false
---

## 結論: jsconfig.json 5行で「Cannot find module」補完落ちは消える

TS未導入のJSプロジェクトでVS Codeの補完が死ぬ原因の9割は、ルートに`jsconfig.json`が無いためエディタが`baseUrl`とモジュール解決方式を推測できないことにある。tsconfigへ移行せず、以下5行で`import`補完とエイリアス解決が復活する。

```json
// jsconfig.json (プロジェクトルート)
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "baseUrl": ".",
    "checkJs": false
  },
  "include": ["src/**/*"]
}
```

`moduleResolution: "Bundler"`はVite 5/TypeScript 5.0以降の値。`"node"`のままだと拡張子なしimportで補完が出ない。

## エラー「Cannot find module '@/utils' or its corresponding type declarations」をpaths 1行で潰す

Viteの`resolve.alias`を設定済みなのにエディタだけ赤線が出るのは、`vite.config.js`とjsconfig.jsonでエイリアス定義が二重管理になっているため。jsconfigに`paths`を追記すると赤線が消える。

```jsonc
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]   // vite.config.js の alias と必ず一致させる
    }
  }
}
```

`baseUrl`が無い状態で`paths`だけ書くと`Non-relative paths are not allowed when 'baseUrl' is not set`が出る。`baseUrl: "."`はセット必須。

## checkJs: true で出る「Parameter 'req' implicitly has an 'any' type」をJSDocで黙らせる

補完を強化しようと`checkJs: true`にした瞬間、既存JSに型エラーが大量発生する。全部直さず、関数だけJSDocで型を付けると`any`警告が止まる。

```js
/**
 * @param {import('express').Request} req
 * @param {import('express').Response} res
 */
function handler(req, res) {
  res.json({ ok: true }); // req.query 補完が効くようになる
}
```

ファイル単位で抑止したい場合は先頭に`// @ts-nocheck`、行単位なら`// @ts-expect-error`を使い、段階的にJSDocへ置換する。

## jsconfig→tsconfig 移行で「File is a CommonJS module; it may be converted」を無事故で消す順序

混在エラーを避ける切り替え順は固定で、(1)`jsconfig.json`を`tsconfig.json`にリネーム、(2)`allowJs: true`と`checkJs: false`を追加、の2手だけ先に行う。これで既存JSを残したままTSファイルを追加できる。

```jsonc
// tsconfig.json (jsconfig からリネーム直後)
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "allowJs": true,     // .js を残したまま .ts を共存
    "checkJs": false,    // JS への型チェックは段階的にON
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  },
  "include": ["src/**/*"]
}
```

`allowJs: true`を入れずに`.ts`を足すと、JSからのimportで`Could not find a declaration file for module './foo.js'`が出る。

## tsconfig だけだとViteが.tsをトランスパイルしない「Failed to resolve import」の検証

エディタは緑でもブラウザで`Failed to resolve import "./foo.ts"`が出るのは、tsconfigはエディタ用でありViteのバンドルとは別系統だから。`npm run build`で実体を確認する。

```bash
# 移行後の動作確認 (Vite 5)
npx vite build --logLevel info 2>&1 | grep -E "resolve|transform"
# 出力に "Failed to resolve" が無ければ ESM 解決OK
```

tsconfigの`paths`はVite本体には伝わらない。ビルド時のエイリアスは`vite.config.js`の`resolve.alias`が正、jsconfig/tsconfigはあくまでエディタ補完用、と役割を分離して二重定義の食い違いを断つ。
