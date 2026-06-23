---
title: "package.json 3大地雷：ESM/CJS混在・ピア依存衝突・プライベートレジストリで副業ツール7本が同時死した話"
free: false
---

## 地雷①：`type: "module"` 1行で `require()` が 7本中 5本クラッシュした実測記録

副業ツール群の共通依存を整理しようと `package.json` に `"type": "module"` を追加した。`npm install` は通過した。しかし起動確認で 7本中 5本が即死した。

```
Error [ERR_REQUIRE_ESM]: require() of ES Module /path/to/utils.js not supported.
```

Node.js は `type: "module"` を検出した瞬間、同ディレクトリ以下の `.js` を ESM として評価する。既存の `require()` 呼び出しは全滅する。死んだのは追加前から動いていた CJS スクリプト群だ。

```json
// Before（動作していた）
{
  "main": "src/index.js"
}

// After（7本中5本死亡）
{
  "type": "module",
  "main": "src/index.js"
}
```

## ESM/CJS 共存の正解：`.cjs` 拡張子分離と `exports` フィールド二重定義

修正パスは 2 通りある。

**パス A：拡張子で明示分離（副業ツールの即戦修復に向く）**

```bash
# CJS のままにしたいファイルを .cjs にリネーム
mv src/legacy-helper.js src/legacy-helper.cjs

# インポート側を一括書き換え
sed -i "s/require('.\/legacy-helper')/require('.\/legacy-helper.cjs')/g" src/*.js
```

**パス B：`exports` フィールドで条件付きエクスポート（ライブラリ配布向け）**

```json
{
  "type": "module",
  "exports": {
    ".": {
      "import": "./src/index.mjs",
      "require": "./src/index.cjs"
    }
  }
}
```

死亡した 5本はパス A で 30 分以内に全復旧した。パス B はインポート元が外部ライブラリとして使う場合に採用する。

## 地雷②：npm 7+ で `@anthropic-ai/sdk + LangChain` がピア衝突する連鎖パターン

npm 7 以降、ピア依存が自動インストールされる仕様に変わった。`@anthropic-ai/sdk` と `langchain` を同居させると次のエラーが出る環境がある。

```
npm error ERESOLVE unable to resolve dependency tree
npm error peer typescript@">=4.9.0 <5.0.0" from @langchain/core@0.3.x
npm error   Found: typescript@5.4.5
```

`--legacy-peer-deps` で黙らせると動くが、後続の `npm install <別パッケージ>` で依存ツリーが再計算されて別の衝突が出る。一時しのぎにしかならない。

正解は `overrides` で明示固定する。

```json
{
  "overrides": {
    "typescript": "5.4.5"
  },
  "dependencies": {
    "@anthropic-ai/sdk": "^0.26.0",
    "langchain": "^0.2.0"
  }
}
```

```bash
# overrides 反映後は lockfile ごと再生成
rm -f package-lock.json
npm install

# 1バージョンに統一されたか確認
npm ls typescript
# └── typescript@5.4.5  ← 複数行出なければ解消
```

## 地雷③：`.npmrc` スコープ漏れで会社 PC だけ `E404` になる構造

社内に Artifactory / Nexus 等のプライベートレジストリがある環境では、`@company/` スコープのパッケージが public npm registry に向かって 404 を返す。

```
npm error code E404
npm error 404 Not Found - GET https://registry.npmjs.org/@company%2finternal-lib
```

プロジェクトルートの `.npmrc` にスコープとレジストリを対応付ければ解決する。

```ini
# .npmrc
@company:registry=https://artifactory.example.com/artifactory/api/npm/npm-local/
//artifactory.example.com/artifactory/api/npm/npm-local/:_authToken=${NPM_TOKEN}
```

```bash
# トークンを環境変数へ（.bashrc / .zshenv に追記）
export NPM_TOKEN="your-token-here"

# 再実行
npm install
```

逆のパターンもある。副業ツールは public パッケージしか使わないのに、`.npmrc` が社内設定を継承して private レジストリへのみ向く場合だ。その場合はプロジェクトスコープで上書きする。

```ini
# .npmrc（副業ツール用リポジトリ）
registry=https://registry.npmjs.org/
```

## env-diag.js 障害 #9・#10 の修復完了確認

本章の作業が終わったら env-diag.js を再実行して `#9` と `#10` の状態を確認する。

```bash
node env-diag.js 2>&1 | grep -E "#9|#10"
```

期待する出力：

```
[PASS] #9  ESM/CJS_MISMATCH  : no require() calls in .mjs files detected
[PASS] #10 PEER_CONFLICT     : typescript resolved to single version (5.4.5)
```

`FAIL` が残る場合、出力末尾に `→ 修復章: Ch.X` が印字される。その章番号で本書を逆引きして再修復する。`PASS` が 2 行揃えば package.json 起因の障害はすべて解消した状態になる。
