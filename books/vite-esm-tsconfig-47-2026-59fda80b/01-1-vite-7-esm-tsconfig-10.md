---
title: "第1章: Vite 7+ESMで頻出するtsconfigエラー10件の逆引き早見表(無料)"
free: true
---

第1章を執筆し、`chapter1_vite_tsconfig.md` に保存した。

**構成の要点（章設計の意図）**

- **冒頭H2前に結論を1文先出し** — 「エラー文字列を Ctrl+F で引いて最小diffを当てるだけで潰せる状態になる」と章末成果を宣言。
- **逆引き早見表を先頭に配置** — 10件すべてを「エラー文字列 → 該当設定キー」で1テーブルにし、unique_angle（エラー文を見出しにした逆引き）を冒頭から体現。
- **H2 5本、全てに固有名詞＋数値** — `Vite 7`／実エラー文字列（`Cannot find module...`、`The current file is a CommonJS module`、`Relative import paths need...`、`verbatimModuleSyntax`）を見出しにそのまま採用。検索KWを見出しに3つ以上含む方針を満たす。
- **各H2に実行可能diff** — `tsconfig.json` の修正前後を `diff` ブロックで提示。各diffに「なぜ直るか」を1行添付（例: `Bundler`は`exports`を辿る、`allowImportingTsExtensions`は`noEmit`とセット必須）。
- **章末の購買動機** — 残り37件を具体名（`vite-tsconfig-paths`連携、Vitest の `types` 衝突、CIだけ落ちる差分）で列挙し、「47件を自動分類してdiffを当てるBashスクリプトは第2章以降」と、続きを買う動機を自然に提示。

**自己点検**: 全H2にコードブロック有り／AI常套句（私は・思います・ぜひ・皆さん等）なし／各見出しに数値or固有名詞有り／unique_angle（エラー文＝見出しの逆引き）反映済み。約1200字。
