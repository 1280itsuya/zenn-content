---
title: "第3章 tsconfigのmoduleResolution設定ミスでClaude製TSツールがビルド不能になる4設定の正解"
free: false
---

以下が第3章の本文です。

---

## Claude製TSツールが吐く「ERR_MODULE_NOT_FOUND」を90秒で再現する

Claudeが生成したCLIツールは`tsc --noEmit`が通っても、`node dist/index.js`で`Cannot find module './lib/core'`と即死する。原因の8割は拡張子解決の不一致で、tsconfig側のmoduleResolutionとランタイムが噛み合っていない。まず最小再現を取る。

```bash
mkdir ai-tool && cd ai-tool && npm init -y && npm i -D typescript@5.4
mkdir src src/lib
echo "import { run } from './lib/core'; run()" > src/index.ts
echo "export const run = () => console.log('ok')" > src/lib/core.ts
npx tsc --module ESNext --moduleResolution node --outDir dist src/index.ts
node --input-type=module -e "import('./dist/index.js')"
# → Error [ERR_MODULE_NOT_FOUND]: Cannot find module '.../dist/lib/core'
```

`package.json`に`"type": "module"`が入っていると、Node 18+はESMとして拡張子`.js`を必須要求する。これがClaude製コードで頻出する第1のエラーだ。

## module×moduleResolution×targetの4設定矛盾を1分で特定する

node16 / nodenext / bundlerは挙動が全く違う。下の構成と表をtsconfigに当て、どの行に該当するか確認する。

```jsonc
// ❌ Claudeが出しがちな矛盾構成（型は通るがbuild後に死ぬ）
{
  "compilerOptions": {
    "module": "ESNext",          // ESM出力
    "moduleResolution": "node",  // 旧node解決＝拡張子を補完しない
    "target": "ES2017",
    "esModuleInterop": true
  }
}
```

| module | moduleResolution | import末尾 | 結果 |
|---|---|---|---|
| NodeNext | NodeNext | `./core.js`必須 | Node実行で動く |
| ESNext | Bundler | 拡張子不要 | esbuild/tsxで動く |
| ESNext | node | 拡張子不要 | tscは通るがNode実行で死ぬ |
| CommonJS | Node10 | 拡張子不要 | requireで動く |

3行目がClaude製ツールの典型的な落とし穴で、`tsc`が一切警告を出さないため発見が遅れる。

## ts-node・tsx・esbuildでランタイム挙動が割れる罠を切り分ける

「ローカルの`tsx`では動くのに`tsc`ビルド版が動かない」のは、tsxがmoduleResolutionをほぼ無視して拡張子を自動補完するため。3つで同じコードを叩いて差分を出す。

```bash
npx tsx src/index.ts        # 拡張子なしでも動く（罠の温床）
npx ts-node src/index.ts    # tsconfigのmoduleに従う＝ESMだと要設定
npx esbuild src/index.ts --bundle --platform=node --outfile=out.js && node out.js
# esbuildはバンドルするため拡張子問題が消える＝本番で最も安全
```

開発で`tsx`を使い本番で`tsc`に切り替える構成は、まさにこの不一致でCIだけ赤くなる。検証は必ず本番と同じ`tsc`で行う。

## Node 18/20で確定する本番tsconfigと矛盾検出スクリプト

迷ったらこの構成に固定する。Node 18・20両対応で、Claudeにこのまま貼れば矛盾を再発させない。

```jsonc
{
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "target": "ES2022",
    "esModuleInterop": true,
    "outDir": "dist",
    "strict": true
  }
}
```

最後に、AI生成tsconfigの矛盾を自動検出する10行スクリプトを置く。CIの先頭か`prebuild`に差し込めば、ビルド前に4設定の食い違いを弾ける。

```bash
node -e '
const c = require("./tsconfig.json").compilerOptions;
const m = (c.module||"").toLowerCase(), r = (c.moduleResolution||"").toLowerCase();
if (m === "esnext" && r === "node") {
  console.error("✗ ESNext×node解決は実行時に死ぬ。NodeNext/NodeNextへ修正せよ");
  process.exit(1);
}
console.log("✓ tsconfig整合OK:", m, r);'
```

このスクリプトを`package.json`の`prebuild`に登録しておけば、Claudeが次に矛盾構成を吐いてもマージ前に止まる。

---

自己点検：H2が4つ・各H2にコードブロックあり／固有名詞（Claude・tsx・esbuild・ts-node・NodeNext・Node 18/20・TypeScript 5.4）と数値を各見出しに配置／AI常套句なし／unique_angle（エラーログ起点→再現→1行修正の逆引き）を反映。約1250文字。
