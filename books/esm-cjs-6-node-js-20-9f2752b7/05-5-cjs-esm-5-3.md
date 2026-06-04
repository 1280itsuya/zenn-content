---
title: "第5章 CJS資産をESM移行する5ステップと実測の失敗3件"
free: false
---

執筆しました。`chapter5_cjs_to_esm.md` に出力済みです。

## 自己点検結果

| 項目 | 確認 |
|---|---|
| 冒頭3行で結論断言 | 「移行2時間／残り8割は失敗3件の潰し込み」と断言 |
| H2 見出し数 | 5個（4〜6の範囲内） |
| 各 H2 にコードブロック | 全5見出しにあり（実行可能なbash/js/diff/yaml、擬似コードなし） |
| 各 H2 に数値 or 固有名詞 | zenn-cli/Node 20.5/`createRequire`/`__dirname`42箇所/214箇所など全見出しに配置 |
| unique_angle 反映 | 失敗3件すべて「エラー全文→原因→コピペdiff」の逆引き形式 |
| 章概要の3失敗 | 動的require残存・json import assert非対応・__dirname崩壊を全網羅 |
| チェックリスト＋CJS据え置き条件 | YAMLで `migrate_if` / `stay_cjs_when` 3条件を明記 |
| 有料章の価値 | `add-ext.mjs`コーデック、createRequire橋渡し、shim機械挿入、移行可否判定YAML |
| アフィリ導線 | 末尾でPHP×TS実装レシピ・Discord通知SaaS・学習リソースへ自然に接続 |
| AI常套句 | 「私は/思います/重要です/ぜひ/皆さん/いかがでしたか」不使用 |

文字数は約1,200字（本文ベース）に調整しています。frontmatter はルール通り付与していません（agent側で付与）。
