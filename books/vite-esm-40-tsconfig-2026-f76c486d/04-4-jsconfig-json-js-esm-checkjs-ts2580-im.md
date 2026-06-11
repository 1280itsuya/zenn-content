---
title: "第4章 jsconfig.json勢の罠:型なしJS+ESMで\"checkJs\"を段階導入しTS2580/import.meta事故を消す"
free: false
---

第4章を執筆しました。`chapter4_jsconfig_checkjs.md` に保存済みです。

**章設計の要点:**

- **目的の1文**: jsconfig運用のまま `@ts-check`→ファイル単位→jsconfig段階適用の順序でTS事故を消す
- **章末の成果物**: 4件の git diff + jsconfig→tsconfig 最小変換表(課金者特典として明示)
- **unique_angle 反映**: 全H2が「実エラー全文 → git diff 1個」の逆引き辞書形式。概念解説ゼロ、検索流入が該当エラーへ即着地できる構成

**ルール適合チェック:**

| 項目 | 状態 |
|---|---|
| 小見出し ## 4〜6個 | 5個 ✅ |
| 各見出し下にコードブロック | 全見出しに1〜2個 ✅ |
| 実行可能コード(擬似コード禁止) | jsonc/diff/bash/ts 実コード ✅ |
| AI常套句(私は/思います/皆さん 等) | 不使用 ✅ |
| 各見出しに数値 or 固有名詞 | TS2580/TS2304/TS17004/420件/import.meta/esbuild/Vite ✅ |
| 一般語彙(はじめに/まとめ)回避 | 回避 ✅ |
| 有料章の課金価値 | 最終H2の変換表を「課金者特典」として配置 ✅ |
| 文字数 | 約1,200字相当 ✅ |

frontmatter は付けていません(agent側付与の指示どおり)。
