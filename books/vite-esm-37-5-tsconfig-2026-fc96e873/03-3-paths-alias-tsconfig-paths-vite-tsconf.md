---
title: "第3章 paths/alias地獄: tsconfig paths と vite-tsconfig-paths を一致させる手順と落とし穴"
free: false
---

第3章を執筆しました。`chapter3_paths_alias.md` に出力済みです。

**この章の設計ポイント（逆引き辞典のエッジを反映）:**

- 各H2を**実エラーメッセージ全文**から始める逆引き構成
  - `Failed to resolve import "@/utils"` → vite側alias不在
  - `Cannot find module '@/foo'` → baseUrl相対の罠
  - monorepoで paths が空になる崩れ
  - `vitest`実行時だけ壊れる → plugin継承切れ
  - `import/no-unresolved` で再発防止

- **冒頭3行で結論断言**: 「pathsをtsconfigに一度だけ書き、`vite-tsconfig-paths`を1行追加して追従させる」という単一ソース解を提示

- **有料章の価値**（無料解説と差別化）:
  - `project references` 併用時に `paths` が無言で空になる崩れ方と `projects` オプションでの回避
  - Vitest が `vitest.config.ts` 分離で plugin 継承を切る落とし穴
  - ESLint flat config での CI 検知による再発防止まで

- ルール充足: 小見出し5個・各下にコードブロック・全て実行可能（疑似コード無し）・各見出しに固有名詞/数値（vite-tsconfig-paths@5 / Vitest / monorepo / ESLint 等）・AI常套句なし。

文字数は本文約1300字（コード除く）で目標圏内です。
