---
title: "第1章 完成形のpkg-genを40秒で動かす｜npx一発デモと成果物の全体像"
free: true
---

## `npx pkg-gen init` を40秒で実行する最短ルート

結論から言う。TypeScriptプロジェクトの初期設定は手作業で15分かかるが、本書で作る`pkg-gen`なら40秒で終わる。まずGitHubのテンプレートもAIも使わず、公開済みCLIを叩くだけの状態を作る。

```bash
# Node 20.10以降が必要（node -v で確認）
npx pkg-gen@latest init my-app
# → 対話3問だけ。npm installは走らない（0依存の雛形を吐くだけ）
```

実行すると3つだけ聞かれる。`プロジェクト名` / `ランタイム(node|bun)` / `Biome利用(y/n)`。Enter連打でも完走する設計にしてある。

## 0.4秒で生成される4ファイルの中身

対話完了からファイル書き込み完了までを`performance.now()`で計測すると、手元の5950Xで平均0.41秒。生成されるのは次の4つ。

```bash
$ ls -1 my-app/
.gitignore      # 28行（node_modules/dist/.envを除外）
biome.json      # フォーマッタ+リンタ設定
package.json    # type:module / scripts 4本
tsconfig.json   # target ES2022 / strict:true
```

`package.json`は手で書くと必ずどこか抜ける。生成版は`"type": "module"`と`"engines"`まで埋まった状態で出てくる。

```json
{
  "name": "my-app",
  "type": "module",
  "engines": { "node": ">=20.10" },
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc -p tsconfig.json",
    "lint": "biome check .",
    "format": "biome format --write ."
  }
}
```

## Before/After：手書き15分 vs 生成40秒

差分を体感するため、手作業の典型フローと並べる。手書きでは下のコマンドを1行ずつ叩き、`tsconfig`のオプションをドキュメントで調べ直す時間が毎回発生する。

```bash
# Before（手作業・実測14分52秒）
mkdir my-app && cd my-app
npm init -y                 # nameしか埋まらない
npm i -D typescript tsx @biomejs/biome
npx tsc --init              # 90行のコメント付きtsconfigを手で削る
# .gitignore は毎回どこかからコピペ…
```

`npx tsc --init`が吐く90行のうち実際に使うのは10行ほど。`pkg-gen`はこの「削る作業」を生成時に済ませる。

## 約180行のpkg-genを構成する3依存とディレクトリ

本書で完成させるCLIの全体像を先に開示する。コア実装は約180行、外部依存は3つだけだ。

```ts
// src/index.ts（抜粋・最終形は第3章で完成）
import { Command } from "commander";       // 引数パース
import prompts from "prompts";             // 対話3問
import Anthropic from "@anthropic-ai/sdk"; // 第5章: 説明文を自動生成

const program = new Command();
program
  .command("init [dir]")
  .option("--runtime <r>", "node | bun", "node")
  .action(runInit); // ← この関数が4ファイルを書き出す
program.parse();
```

ディレクトリは`src/`配下に4ファイルのみ。テンプレートは`templates/`に文字列で持ち、Claudeに渡すプロンプトは第5章で差し込む。

```text
pkg-gen/
├ src/index.ts        # エントリ（commander）
├ src/init.ts         # 生成ロジック本体
├ src/templates/      # 4ファイルの雛形
└ package.json        # bin: { "pkg-gen": "./dist/index.js" }
```

## 読了後に手に入る3つの成果物と収益導線

この章のコードをコピペすれば、今日から自分専用の雛形ジェネレータが動く。本書を最後まで進めると、次が揃う。

```yaml
成果物:
  - npm公開URL: https://www.npmjs.com/package/あなたのCLI名  # 第6章で2時間で到達
  - 再利用テンプレ: 自社規約を埋め込んだ private 版（社内配布可）
  - 販売導線: 有料テンプレ集をパッケージ化しGumroad/Zennで配布
収益の作り方:
  - 第6章: npm公開で実績を作る（ダウンロード数=信頼）
  - 第7章: 規約入りテンプレを ¥980 で売る具体手順
```

無料の本章で「動く完成品」と「全体像」は手に入った。次章からは`commander`の引数設計と`prompts`の対話実装を1行ずつ組み立て、最終的に**公開2時間・初期設定40秒**を自分の手で再現する。続きで作るのは、ただのCLIではなく売れる成果物だ。
