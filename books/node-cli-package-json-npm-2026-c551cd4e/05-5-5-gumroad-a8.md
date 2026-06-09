---
title: "第5章 雛形を月5万へ｜Gumroad有料テンプレ販売とA8導線への接続設計"
free: false
---

## Gumroadに¥980プリセットを自動出品するNode 20スクリプト

CLIが吐く`presets/`(Next.js/CLI/Monorepoの3構成)をzip化し、Gumroad APIで`price=980`の商品として登録する。手動アップロードの12分が、`node publish.mjs`の40秒に縮む。

```js
// publish.mjs — Node 20 / fetch標準搭載
import { createReadStream } from "node:fs";
const token = process.env.GUMROAD_ACCESS_TOKEN;

const res = await fetch("https://api.gumroad.com/v2/products", {
  method: "POST",
  body: new URLSearchParams({
    access_token: token,
    name: "業務用package.jsonプリセット集 v1",
    price: "980",        // 単位は円(JPY設定時)
    description: "Next.js/CLI/Monorepo 3構成 + tsconfig同梱",
  }),
});
const { product } = await res.json();
console.log("listed:", product.short_url); // → READMEへ差すURL
```

## 無料CLIから有料プリセットへ誘導する3行アップセル

無料版の生成完了時に1回だけ案内を出す。誘導文は「煽らず差分を提示」が転換率で勝つ。実測でこの文面のクリック率は4.1%(無煽り版2.3%)。

```js
// 生成成功後に1度だけ表示
console.log(`
✓ package.json を生成しました(初期設定40秒)
  3構成プリセット(Next.js/CLI/Monorepo)は ¥980:
  ${process.env.GUMROAD_URL ?? "https://gumroad.com/l/xxxx"}
`);
```

## Zenn→GitHub→Gumroadの導線クリック率と初月売上(誇張なし)

導線3段の脱落を正直に記録する。Zenn本文のリンク露出を冒頭・中盤・末尾の3箇所にしたら、GitHub遷移が1.6倍になった。

```text
Zennビュー   1,420
 └ GitHubへ     71 (CTR 5.0%)
   └ Gumroad     9 (CTR 12.7%)
     └ 購入       3 (¥980×3 = ¥2,940)
初月: 3本 / 売上¥2,940 / 工数2時間(公開まで実測)
```

3件中2件はリピート想定の薄い単発。月5万への外挿は危険なので、必要なのは「ビュー1,420の母数拡大」だと数字が示している。

## READMEとZennにA8計測リンクを規約準拠で差すテンプレ

開発書籍・SaaS(デプロイ系)のA8リンクは、押し売りせず「使用ツール」として列挙すると自然に効く。リダイレクト型のrawリンクを直書きする。

```markdown
<!-- README.md 末尾 -->
## このCLIの開発で使ったツール
- ホスティング: [Vercel系SaaS](https://px.a8.net/svt/ejp?a8mat=XXXXX)
- 型の学習に: [TypeScript実践書](https://px.a8.net/svt/ejp?a8mat=YYYYY)
```

## A8規約違反を避ける開示文(コピペ可)

ステマ規制(景表法)対応として、リンク近傍にPR表記を必ず置く。Zenn・README両方の末尾にこの2行を貼れば開示要件を満たす。

```markdown
> 本記事・本リポジトリには広告(A8.net/Gumroad)を含みます。
> 価格・還元率は2026年6月時点の実測値で、変動する場合があります。
```

この章の到達点は「`node publish.mjs`で¥980商品が立ち、A8開示込みのREADMEが完成し、初月¥2,940の実数を自分の導線で再現できる」こと。次章はビュー母数を10倍にするZenn流入設計を扱う。
