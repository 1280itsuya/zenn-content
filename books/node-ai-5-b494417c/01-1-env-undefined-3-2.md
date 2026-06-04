---
title: "第1章 無料試読: 『env が undefined』を3分で切り分ける2択フロー"
free: true
---

## `Claude API 401`の8割は鍵切れではなく`process.env`消失

結論を3行で出す。`401 Unauthorized`が返ったら、まず疑うべきは無効なAPIキーではなく`process.env.ANTHROPIC_API_KEY`が`undefined`になっている状態だ。筆者が踏んだ23件のAI副業ツール不動のうち18件（約78%）はこのパターンで、キー自体は生きていた。判定は次の1行で始める。

```bash
node -e "console.log('key len:', (process.env.ANTHROPIC_API_KEY||'').length)"
# => key len: 0  なら鍵切れではなく env 消失。0でなければ別原因
```

`len: 0`なら以降の2択フローへ進む。`108`など正常長が出たのに401なら、本章の対象外（キー失効・組織制限）なので時間を使わない。

## `typeof window`と`typeof process`の1行で発生環境を2択判定

本書共通の武器がこれだ。`env`が消えた現場がNodeかブラウザかを、ツールを起動せず1行で確定させる。

```ts
const env =
  typeof window !== 'undefined' ? 'Browser'
  : typeof process !== 'undefined' ? 'Node'
  : 'Unknown';
console.log('[diag] runtime =', env); // [diag] runtime = Node
```

`Browser`なら後述のVite分岐、`Node`なら`dotenv`分岐へ。所要は2手で約3分。この1行を全エラーログの先頭に常駐させると、5パターンの切り分け初手が固定化される。

## Node側の分岐: `dotenv`のload漏れと実行`cwd`ずれ

`runtime = Node`で`len: 0`の原因は2つに割れる。`dotenv`の読み込み漏れと、実行ディレクトリ（cwd）が`.env`とずれているケースだ。

```bash
# 1. cwd と .env の位置を突き合わせる
node -e "console.log('cwd:', process.cwd()); const fs=require('fs'); console.log('.env exists:', fs.existsSync('.env'))"
# cwd: /app/scripts   .env exists: false  ← 親に .env があるのにここで実行 = cwd ずれ
```

```ts
// 2. import より前に絶対パス指定で読む（require 順序が9割の死因）
import { config } from 'dotenv';
import { resolve } from 'node:path';
config({ path: resolve(__dirname, '../.env') }); // ← Anthropic 初期化より前
```

`exists: false`ならcwdずれ、`true`なのに`len: 0`ならload順序漏れ。修正diffは第2章に収録。

## ブラウザ側の分岐: Viteの`define`未設定で`import.meta.env`が空

`runtime = Browser`なら`.env`は無関係だ。バンドラが環境変数を埋め込んでいないのが真因で、Viteでは`VITE_`接頭辞か`define`設定が抜けている。

```ts
// vite.config.ts — define 未設定だと process.env.* が undefined のまま残る
export default {
  define: { 'process.env.ANTHROPIC_API_KEY': JSON.stringify(process.env.ANTHROPIC_API_KEY) },
};
// 推奨は VITE_ 接頭辞 → import.meta.env.VITE_ANTHROPIC_API_KEY
```

なお、APIキーをブラウザに埋め込むと第三者に露出する。Browser分岐の正解はキー直挿しではなくプロキシ経由で、その実装は第5章で扱う。

## 症状→該当章 対応表で『自分のバグはどれか』を確定

この無料章のゴールは、5パターンのうち自分の症状が「どれで、何章を読むか」を確定させることだ。下表で対応章へ直行できる。

```text
| # | 症状(ログ)               | runtime | 真因               | 修正章 |
|---|--------------------------|---------|--------------------|--------|
| 1 | 401 / key len:0          | Node    | dotenv load漏れ・cwdずれ | 第2章 |
| 2 | 401 / undefined          | Browser | Vite define 未設定 | 第3章 |
| 3 | fetch failed / ECONNRESET| Node    | proxy・timeout設定 | 第4章 |
| 4 | CORS blocked             | Browser | キー直挿し→要proxy | 第5章 |
| 5 | tree-shaking で消える呼出 | Browser | bundler最適化漏れ  | 第5章 |
```

## 次章: 切り分け後のコピペdiffで5パターンを塞ぐ

ここまでで「runtime判定→真因→修正章」の3ステップが3分で回るようになった。残るは手を動かすだけだ。第2章以降は上表の各行に対応した**コピペで貼るだけのdiff**を、`before`/`after`の差分形式で全5パターン分そろえている。自分の症状番号が決まった読者は、該当章のdiffを当てれば沈黙したツールが最短で復帰する。
