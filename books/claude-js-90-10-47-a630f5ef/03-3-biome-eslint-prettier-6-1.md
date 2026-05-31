---
title: "第3章 Biome導入でESLint+Prettier 6設定を1ファイルに統合する手順"
free: false
---

## ESLint+Prettier 6ファイルをbiome.json 1本に畳む

移行前のリポジトリには `.eslintrc.json` `.eslintignore` `.prettierrc` `.prettierignore` `eslint.config.js` `.editorconfig` の6ファイルが散っていた。Biome 1.9系はlintとformatを1プロセスで処理するため、これらを `biome.json` 1ファイルへ統合できる。まず既存設定を退避してBiomeを入れる。

```bash
mkdir -p .legacy-lint && mv .eslintrc.json .prettierrc eslint.config.js .legacy-lint/
npm i -D --save-exact @biomejs/biome@1.9.4
npx @biomejs/biome init   # biome.json を生成
```

`init` が吐く雛形は最小限なので、ここからClaudeに肉付けさせる。

## Claudeにbiome.jsonを生成させる入力プロンプト

退避した6ファイルの中身をそのまま貼り、ルールを欠落させないよう「対応表を出せ」と指示するのが要点になる。

```text
以下のESLint/Prettier設定6ファイルを Biome 1.9.4 の biome.json 1ファイルに統合して。
- 各旧ルール → Biome側ルール名の対応表をMarkdown表で先に出す
- 対応するBiomeルールが無い項目は「未対応」と明記
- formatter.indentWidth, lineWidth は .prettierrc の値を維持
[ここに6ファイルの全文を貼付]
```

対応表を先に出させると、消えたルールが一覧で見えるため検証が10分で済む。

## import順ルールが効かなかった失敗と修正プロンプト

最初の生成では `organizeImports` が `"enabled": true` になっていたのに、保存してもimportが並び替わらなかった。原因は、Biomeのimport整列はESLintの `import/order` とグループ定義の粒度が違い、ルール名を曖昧に渡すと既定値で上書きされる点にある。ルール名を明示して再生成させた。

```jsonc
// 修正後の biome.json 抜粋
{
  "organizeImports": { "enabled": true },
  "linter": {
    "rules": {
      "style": { "useImportType": "error" },     // 旧 @typescript-eslint/consistent-type-imports
      "correctness": { "noUnusedImports": "error" } // 旧 unused-imports/no-unused-imports
    }
  }
}
```

修正プロンプトは「`import/order` 相当を `organizeImports` ＋ `useImportType` で再現し、ルール名を必ずJSON内にコメントで残せ」の1行追加で通った。

## CI時間42秒→9秒：GitHub Actionsの計測

旧構成は `eslint .` と `prettier --check .` を直列で回し42秒。Biomeは `biome ci` 1コマンドでlint+formatを同時実行し9秒に縮んだ（約4.6倍、対象1,800ファイル）。

```yaml
# .github/workflows/lint.yml
- uses: biomejs/setup-biome@v2
  with:
    version: 1.9.4
- run: biome ci .   # lint + format + import整列を一括
```

`npm script` も `"lint": "biome check --write ."` の1行に集約でき、`package.json` のdevDependenciesが8個減った。

## 移行チェック第31〜38項目とJS/TS混在の落とし穴

チェックリスト47項目のうち、本章該当は第31〜38項目。実際に12回の試行で踏んだ罠を数値付きで残す。

```bash
# 第31〜38項目の自動確認スクリプト
biome check . | tee result.txt
grep -c "error" result.txt          # 31: 残存エラー数を0に
test -f .legacy-lint/.eslintrc.json # 38: 旧設定の退避確認
```

- 第33項目: `.js` と `.ts` 混在リポでは `overrides` で `"**/*.js"` に `useImportType: off` を指定しないと、JS側で287件の誤検知が出る
- 第36項目: `.editorconfig` を残すとindentWidthが二重定義になり、`biome migrate` が警告を14件吐く。`biome.json` 側に一本化する
- 第38項目: VS Codeの既定フォーマッタを `biomejs.biome` に切替えないと、保存時にPrettierが復活して差分が再発する

47項目の完全版チェックリストと本章の再利用プロンプトは付録Aの配布ZIPに同梱した。次章はzenn-cliのプレビュー環境を扱う。

---
topics: ["claude","typescript","vite","biome","githubactions"]
