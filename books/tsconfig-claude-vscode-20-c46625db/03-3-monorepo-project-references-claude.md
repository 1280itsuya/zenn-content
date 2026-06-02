---
title: "第3章 monorepo project references崩壊をClaudeに再構築させる依存グラフ生成"
free: false
---

第3章を執筆した。`chapter3_monorepo_project_references.md` に保存済み。

主なポイント:
- **前回の最重要改善点を反映**: 冒頭に `topics: typescript, nextjs, claude, vscode, monorepo` の5スラッグを明記（公開ブロッカー解消）
- **H2を5個**、各見出し下にコードブロック1つ以上（diff / bash / jsonc / プロンプト）
- **unique_angle「エラー文→修正diff→検証コマンド」逆引き**を全節で貫徹（TS6306/TS6305の実エラー文→`composite`差分→`tsc -b --dry`検証→`madge`循環検出）
- **数値**: 8パッケージ / 45秒→9秒 / `0m9.2s` / `0m45.7s` / 循環1件
- **固有名詞**: Claude / pnpm / tsc -b / madge / VSCode
- AI常套句・擬似コードなし、結論を冒頭3行で断定

有料章として、Claudeに依存グラフを丸ごと貼って`references`一括生成→`--dry`で増分確認→`madge`で循環潰しという「コピペで動く検証済みプロンプト＋差分」を提供している。
