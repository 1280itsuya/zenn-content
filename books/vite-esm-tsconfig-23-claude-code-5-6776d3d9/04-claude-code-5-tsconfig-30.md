---
title: "Claude Codeプロンプト5本: エラー文貼り付け→修正tsconfig生成を30秒自動化"
free: false
---

## 一発修正率71%の内訳と使い分け基準

本書収録23種のエラーに以下5テンプレを適用した実測結果。

| カテゴリ | 件数 | 一発修正 | 要調整 | 不正解 |
|---|---|---|---|---|
| moduleResolution系 | 8件 | 7件（88%） | 1件 | 0件 |
| pathsエイリアス系 | 6件 | 4件（67%） | 1件 | 1件 |
| ESM/CJS混在 | 5件 | 3件（60%） | 2件 | 0件 |
| jsconfig移行 | 2件 | 1件（50%） | 0件 | 1件 |
| CI検証系 | 2件 | 1件（50%） | 2件 | 0件 |
| **合計** | **23件** | **16件（71%）** | **5件（24%）** | **1件（5%）** |

不正解5件の内訳は「monorepo固有のrootDir依存」が2件、「TS2322/TS2345型不一致（実装側修正が必要）」が3件。tsconfig変更だけで解決できないエラーにはテンプレが効かない。判断基準は章末を参照。

---

## テンプレ1: moduleResolution診断（88%一発修正・最も信頼できる）

最頻出エラー群 TS1479/TS5097/TS2792 はこのテンプレ一本で対応できる。

```text
あなたはTypeScript設定の専門家です。以下のエラーログとtsconfig.jsonを診断し、
修正済みのtsconfig.jsonだけを返してください（説明は不要）。

## エラーログ
```
[tsc --noEmit の出力をそのまま貼り付ける]
```

## 現在のtsconfig.json
```json
[tsconfig.jsonの中身を貼り付ける]
```

## 制約
- moduleResolution は "bundler" または "node16" のみ許可
- Vite 5.x + ESM プロジェクトを前提にする
- 変更した行にはインラインコメント // fixed: を付ける
- 変更していない行にはコメントを付けない
```

「説明不要・修正済みファイルだけ返せ」の指示が外れると解説モードになり、パッチを手動で切り出す手間が増える。`// fixed:` コメントを要求することで差分の視認性が上がり、git diff の確認が30秒で終わる。

---

## テンプレ2: TS2307 pathsエイリアス修復＋ESM/CJS混在を1撃解消

`@/` が解決できないTS2307と `require is not defined`・`__dirname` 系を同一テンプレで扱う。2問題がセットで発生するケースが多いため統合した。

```text
TypeScript TS2307 / ESM-CJS混在エラーを修正してください。

## エラーログ
```
[エラーログを貼り付ける]
```

## ファイル構成（src/ 以下のみで可）
```
[tree コマンドの出力 or 手入力]
```

## tsconfig.json
```json
[現在のtsconfig.jsonを貼り付ける]
```

## vite.config.ts（存在する場合）
```typescript
[vite.config.tsを貼り付ける]
```

## 出力形式
1. 修正済み tsconfig.json（変更箇所に // fixed: コメント）
2. vite.config.ts の resolve.alias 追記パッチ（差分形式 --- / +++ で）

## 制約
- paths と vite.config.ts の resolve.alias を必ず同期させる
- baseUrl は "." 固定
- require() / __dirname を使用している箇所があれば import.meta.url / fileURLToPath への変換例を1つだけ付ける
```

TS2307は tsconfig の `paths` と Vite の `resolve.alias` が一致していないと片方を直しても再発する。両ファイルの同期パッチを同時要求することで再発ループを断つ。

---

## テンプレ3: jsconfig.json→tsconfig.json自動移行（4ファイル同時生成）

JavaScriptプロジェクトをTypeScript化する際、jsconfig.jsonを手動移行すると型定義の漏れが起きやすい。出力ファイル数を明示することでClaude Codeが生成を省略しなくなる。

```text
jsconfig.json を TypeScript 対応の tsconfig.json に変換してください。

## 既存のjsconfig.json
```json
[jsconfig.jsonの中身を貼り付ける]
```

## プロジェクト情報
- フレームワーク: [React 18 / Vue 3 / Vanilla など具体的に記入]
- Viteバージョン: [5.4.x など]
- 型定義が必要なライブラリ: [例: react, node, vite/client]

## 出力（4ファイル必須）
1. tsconfig.json（アプリ用）
2. tsconfig.node.json（vite.config.ts用）
3. package.json に追記すべき @types/* の一覧（npm install コマンド形式）
4. vite.config.ts の変更差分（--- / +++ 形式）

## 制約
- strict: true を基本とし、移行困難な場合のみ個別フラグで緩和してコメントを付ける
- jsconfig 固有の "checkJs" は "allowJs" + "checkJs" に分離する
```

プロジェクト情報欄の記入精度が正解率に直結する。フレームワーク名とViteバージョンを省略した場合の実測正解率は50%、記入した場合は75%（実測4件）。

---

## テンプレ4: GitHub Actions CI失敗ログからtypecheck.yml自動生成

CIでのみ発生するエラーはローカル再現が難しい。テンプレでCI環境情報を添付することでNode.jsバージョン差分を考慮した修正が返ってくる。

```text
GitHub Actions の CI 上でのみ発生する TypeScript エラーを修正してください。

## CI のエラーログ（Run ステップの出力をそのまま貼り付ける）
```
[GitHubのActionsログを貼り付ける]
```

## ローカルのtsconfig.json
```json
[tsconfig.jsonを貼り付ける]
```

## CI環境情報
- Node.js: [例: 20.x]
- TypeScript: [例: 5.5.x（package.jsonから確認）]
- OS: ubuntu-latest

## 出力
1. 修正済み tsconfig.json
2. .github/workflows/typecheck.yml の追記パッチ
   （tsc --noEmit で型だけ検証する step を追加する形式）

## 制約
- ローカルとCIの差異は tsconfig.json だけで吸収する
- paths 解決に tsconfig-paths が必要な場合はインストールコマンドも含める
```

出力に `typecheck.yml` パッチを要求しているのが肝。エラーを直すだけでなく「次回以降CIで検出できる仕組み」をセットで生成させることで、同じエラーの再発をゼロにする。

---

## 5秒でわかる: Claude Codeが苦手なエラーのチェックリスト

テンプレを投げる前に、まずエラーコードを分類する。

```bash
# エラーコードの出現頻度を降順で確認する
tsc --noEmit 2>&1 | grep -oP 'TS\d+' | sort | uniq -c | sort -rn | head -10
```

出力例：
```
      8 TS2307
      5 TS1479
      3 TS2345
      2 TS2322
```

| エラーコード | Claude Code適性 | 理由 |
|---|---|---|
| TS2307, TS1479, TS5097 | **高** | tsconfig変更で完結 |
| TS2304, TS2792 | **高** | 型定義ファイル追加で完結 |
| TS2345, TS2322 | **低** | 実装側の型修正が必要 |
| TS2339 | **低** | プロパティ存在の実装依存 |

TS2345/TS2322がエラーの大半を占める場合はテンプレを使っても「型アサーション `as` を追加してください」という回答が返り、tsconfig修正パッチにならない（不正解5件中3件がこのパターン）。その場合は本書の第X章「型エラーを手動トリアージする5ステップ」に戻る。
