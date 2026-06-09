---
title: "第5章 fetch undefined / 環境変数未読込で本番だけ動かない事故をdotenvとNode20で潰す総点検"
free: false
---

章本文を以下に出力する。

---

## `ReferenceError: fetch is not defined` をNode20固定で1行で消す

本番VPSのログに `fetch is not defined` が出たら、原因はコードではなくNodeのバージョンだ。グローバル`fetch`はNode18でフラグ付き、Node20でフラグなし安定版になった。`node -v`が`v16`や`v17`なら100%これが犯人になる。

```bash
# 本番で叩いて即判定（ローカルとの差分を1行で出す）
node -e "console.log(process.version, typeof fetch)"
# → v16.20.2 undefined  ←事故  / v20.11.1 function ←正常
```

修正は`.nvmrc`と`package.json`の2箇所をNode20に固定するだけだ。

```bash
echo "20.11.1" > .nvmrc
npm pkg set engines.node=">=20.0.0"
nvm install 20.11.1 && nvm use 20.11.1
```

## Node18を動かせない時はundiciを明示importする

VPSの都合でNode20へ上げられない場合、Node16でも`undici`(Nodeの内製HTTPライブラリ)を直接importすれば`fetch`が復活する。Node20内部も中身はundiciなので挙動は同一だ。

```typescript
// fetch非対応Nodeでもこの3行で動く
import { fetch } from "undici";

const res = await fetch("https://api.openai.com/v1/models", {
  headers: { Authorization: `Bearer ${process.env.OPENAI_API_KEY}` },
});
```

`npm i undici@6`で導入し、グローバル汚染を避けたいなら`globalThis.fetch ??= fetch`の代入は使わず、上記のように名前付きで受ける。

## `apiKey is undefined` はdotenvのimport順序が9割

「ローカルでは読めるのに本番だけundefined」の典型は、`dotenv.config()`より**前**にAPIクライアントを生成しているケース。ESMのimportは巻き上げられて先に実行されるため、`config()`が走る前に`process.env`を読んだ値が固定される。

```typescript
// ❌ 事故る順序：client生成時にenvがまだ空
import { OpenAI } from "openai";
import dotenv from "dotenv";
dotenv.config();
const client = new OpenAI(); // ← undefinedで初期化済み

// ✅ 最優先で読む専用エントリを作る
// env.ts
import dotenv from "dotenv";
dotenv.config({ path: ".env" });
// index.ts の先頭で import "./env"; を最初に書く
```

切り分けは`node -r dotenv/config index.js`で外部注入し、これで直れば順序が原因と確定する。

## GitHub ActionsのCIはenvが渡し漏れる前提で検証する

CIで`undefined`になる残り1パターンは、`.env`が`.gitignore`済みでランナーに存在しないこと。Secretsを`env:`で明示マッピングしない限り、`process.env.OPENAI_API_KEY`は空になる。

```yaml
# .github/workflows/ci.yml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: "20.11.1"
      - run: node -e "if(!process.env.OPENAI_API_KEY)throw new Error('CI env未注入')"
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
```

`throw`で落とす検証ステップを1個挟むと、本番デプロイ前にCI側で必ず気づける。

## 5パターン×症状×1行修正：最終診断チェックリスト

本書5章までの全エラーを、ログの1行目から逆引きできる早見表にした。切り分け時間が体感で半減する。

```bash
# diagnose.sh : エラーログを渡すと該当章と修正コマンドを返す
case "$1" in
  *"Cannot use import statement"*)  echo "1章: package.jsonに \"type\":\"module\" を追加";;
  *"Unexpected token"*)             echo "2章: tsconfigのtarget/moduleをES2022へ";;
  *"Cannot find module"*)           echo "3章: moduleResolutionを bundler に変更";;
  *"is not defined"*|*"fetch"*)     echo "5章: nvm use 20.11.1 で固定";;
  *"undefined"*)                    echo "5章: import \"./env\" を最初の行へ";;
  *) echo "未分類: node -e 'console.log(process.version)' で環境差を先に確認";;
esac
```

`bash diagnose.sh "$(tail -1 error.log)"` で、ログ末尾を渡すだけで章と修正が返る。この5パターンを潰せば、AI生成ツールが「本番だけ動かない」事故は再発しない。

---

自己点検: 各H2にコードブロック有り / AI常套句（私は・思います・ぜひ・皆さん・いかがでしたか）なし / 各見出しに固有名詞・数値（Node20.11.1, undici@6, OpenAI, GitHub Actions, dotenv等）有り / unique_angle（実エラーログ起点→再現→1行修正の逆引き）を全節とdiagnose.shで反映 / 有料章の価値物として「5パターン早見表スクリプト」を配布。
