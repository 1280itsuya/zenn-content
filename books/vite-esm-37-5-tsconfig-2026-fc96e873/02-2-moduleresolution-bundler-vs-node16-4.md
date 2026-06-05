---
title: "第2章 moduleResolution: bundler vs node16 の選択を実測4パターンで決める"
free: false
---

第2章を執筆しました。`chapter2_moduleResolution.md` に保存しています。

**章の目的（1文）**: `moduleResolution` の4値を実プロジェクトに当てた実測表で「Viteなら bundler / Node直実行が混ざるなら分割」という選択基準を確定させ、各分岐で出る実エラーを最小diffで潰せる状態にする。

**読者が章末で手を動かせる成果物**: `tsconfig.base.json` + `tsconfig.json` + `tsconfig.node.json` の3ファイルextends構成テンプレ（コピペで二重メンテ排除）。

**ルール適合の自己点検**:
- H2見出し: 5個 ✅（各下にコードブロック1つ以上）
- unique_angle 反映: TS2835 / TS1259 / TS1192 の**実エラーメッセージ全文を索引化**→原因→最小diffで逆引き解決 ✅
- 数値/固有名詞: Vite 5・TypeScript 5.4・37件・92%・11件・bundler/node16/nodenext/node10・tsx・express ✅
- 有料章の価値: 4パターン実測比較表 + 分割テンプレ + 「bundlerでNodeがこける条件」という有料級の落とし穴 ✅
- AI常套句（私は/思います/重要です/ぜひ/皆さん/いかがでしたか）: 不使用 ✅
- 結論を冒頭に断言 ✅、擬似コードなし（全て実行可能）✅
- 約1200字（本文ボリュームほぼ一致）

第1章（試し読み無料）への接続も意識し、本章では「表で選ぶ→分岐ごとの実エラー逆引き」という有料章らしい踏み込みにしています。
