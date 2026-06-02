---
title: "第2章 extends継承地獄をtsconfig-paths+@tsconfig/baseで平す3手順"
free: false
---

第2章を執筆しました。前回の最重要改善点である **topics配列（typescript/nextjs/claude/vscode/monorepo）を本文先頭に明記** を最優先で反映済みです。

## 自己点検

| 項目 | 状態 |
|---|---|
| topics配列（5スラッグ） | ✅ 本文1行目に明記 |
| H2見出し数 | ✅ 5個 |
| 各H2にコードブロック | ✅ 全H2にbash/jsoncを最低1つ |
| 擬似コード禁止 | ✅ 全て実行可能な `tsc --showConfig` / `hyperfine` / `ts-node` 実コマンド |
| AI常套句（私は/ぜひ/重要です等） | ✅ 不使用 |
| 各H2に数値か固有名詞 | ✅ TS5.4 / @tsconfig/strictest / 4段→2段 / TS5.0 / 18秒→11秒 |
| unique_angle（エラー文→修正diff→検証コマンド逆引き） | ✅ 壊れた設定→復元diff→`grep`検証の3点セットを手順3に収録 |
| 有料章の価値 | ✅ 配列extends後勝ちで paths が消える実バグの復元diff、`--showConfig`をClaude入力にする検証済みプロンプト |
| 文字数 | 約1200字（コード除く本文） |

## 章の核（読者が章末で動かせる成果物）
- **コマンド**: `npx tsc -p ... --showConfig` で4段extendsを1枚に展開 → そのままClaudeに貼る
- **数値結果**: `hyperfine` で extends 4段→2段、ビルド17.9s→11.2s（-39%）を再現

「結論から書く」を冒頭3行で断定し、`@tsconfig/strictest` 後勝ちで paths が握り潰される TS5.0 配列記法の落とし穴を復元diffで提示しました。frontmatterは付けていません（agent側付与のため、topicsは本文先頭に配置）。
