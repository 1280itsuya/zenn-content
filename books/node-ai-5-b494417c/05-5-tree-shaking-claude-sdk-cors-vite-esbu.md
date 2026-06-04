---
title: "第5章 tree-shakingでClaude SDKが消える/CORS壁: Vite・esbuild本番ビルドの罠"
free: false
---

本番ビルドだけで壊れる故障は、原因が「Node実行時」ではなく「esbuild/Viteの静的解析時」に発生する。Viteで`vite build`した瞬間にClaude SDKの初期化が消え、ブラウザ直叩きでCORSが壁になる。この2件を第1章の切り分け表に接続し、5パターン全ての最小diffで章を閉じる。

## tree-shakingでClaude SDKのsideEffects初期化が消える症状

`new Anthropic()`の前段で動くはずの計測登録やpolyfill読み込みが、`vite build`後だけ実行されない。原因はesbuildが「戻り値が使われない副作用コード」を未使用と判定して削除するため。`vite build --minify false`で出力を覗くと、該当行が丸ごと消えている。

```bash
# dev では動くが build では消える行を特定する
npx vite build --minify false
grep -n "registerTelemetry" dist/assets/*.js   # 出力に無ければ削除されている
```

## package.jsonのsideEffectsと__PURE__注釈で削除を止める

自作ラッパーパッケージ側で`sideEffects`を明示すると、その2ファイルだけはtree-shaking対象外になる。逆に意図的に消したい初期化呼び出しには`/*@__PURE__*/`を付けて選別する。

```json
{
  "name": "my-claude-wrapper",
  "sideEffects": ["./src/init.ts", "./src/polyfill.ts"]
}
```

```ts
// 消さない初期化（sideEffectsで保護した）
import "./init";
// 消えても無害な計測は明示的にPURE指定
const span = /*@__PURE__*/ createSpan("boot");
```

## ブラウザ直叩きで429ではなくstatus 0でCORSが詰まる構造

`fetch("https://api.anthropic.com/v1/messages")`をブラウザから呼ぶと、ステータス0・本文空で失敗する。Networkタブのpreflight `OPTIONS`が赤い。これはNode側の権限問題ではなく、ブラウザのCORS制約。APIキーがバンドルに露出する重大事故でもあるため、直叩き自体を消す。

```ts
// これはブラウザでは必ずCORSで死ぬ（かつキー露出）
const res = await fetch("https://api.anthropic.com/v1/messages", {
  method: "POST",
  headers: { "x-api-key": import.meta.env.VITE_KEY }, // 露出NG
});
console.log(res.status); // 0 が返る
```

## Cloudflareサーバーレス関数1枚を噛ませるプロキシで解決する

Vercel/Cloudflareのedge関数を1ファイル挟むと、CORSもキー露出も同時に消える。ブラウザは自ドメインの`/api/chat`だけを呼ぶ。

```ts
// api/chat.ts （サーバー側でキーを保持、CORSヘッダを返す）
export default async function handler(req: Request) {
  const r = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: {
      "x-api-key": process.env.ANTHROPIC_KEY!, // サーバーだけが持つ
      "anthropic-version": "2023-06-01",
      "content-type": "application/json",
    },
    body: await req.text(),
  });
  return new Response(await r.text(), {
    headers: { "access-control-allow-origin": "https://your.app" },
  });
}
```

## 5パターン最小diffと再発防止チェックリスト12項目

第1章の切り分け表へ立ち返ると、5件は「Node実行時の3件」と「ビルド/ブラウザ時の2件」に二分される。第5章で扱った後者2件の最小diffがこれだ。

```diff
- headers: { "x-api-key": import.meta.env.VITE_KEY }   // CORS死＋キー露出
+ const res = await fetch("/api/chat", { method: "POST", body })  // 自ドメイン
```

```yaml
# 再発防止チェックリスト12項目（CIで機械検査できる順）
checklist:
  - vite build --minify false で消失行を grep 検査
  - package.json sideEffects に init/polyfill を列挙
  - 不要計測に /*@__PURE__*/ を付与
  - import.meta.env.VITE_* に APIキーを置かない
  - ブラウザ fetch 先は自ドメイン /api 配下のみ
  - edge関数で anthropic-version: 2023-06-01 を固定
  - access-control-allow-origin を本番ドメインに限定
  - preflight OPTIONS を 200 で返す
  - res.status===0 を CORS失敗として監視に送出
  - Node 3件(第1〜4章)とビルド2件(第5章)を表で再確認
  - dist バンドルサイズ差分を ±5% で監視
  - キー露出は git-secrets でコミット前に遮断
```
