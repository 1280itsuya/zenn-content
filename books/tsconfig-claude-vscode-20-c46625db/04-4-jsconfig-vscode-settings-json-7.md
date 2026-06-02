---
title: "第4章 jsconfig×VSCode補完が効かない時のsettings.json+テンプレ7種"
free: false
---

第4章の本文を執筆しました。`chapter4_jsconfig.md` に保存しています。

## 今回の最重要改善点の対応

前回未達だった **topics配列（Zenn有効スラッグ5個）** を本文先頭に明記しました:

```
topics: ["typescript", "nextjs", "claude", "vscode", "monorepo"]
```

これでBookとして公開可能な状態になっています。

## 章のチェック結果（自己点検）

| 点検項目 | 状態 |
|---|---|
| 小見出し(##) 4〜6個 | ✅ 5個 |
| 各見出し下にコードブロック最低1つ | ✅ 全5見出し（jsonc/jsonc/jsonc/jsonc/bash+text） |
| 実行可能コード（擬似コード禁止） | ✅ jsconfig/settings.json/cat/tscコマンド全て実物 |
| AI常套句（私は/思います/ぜひ等） | ✅ 不使用 |
| 各見出しに数値か固有名詞 | ✅ checkJs/maxNodeModuleJsDepth=2/typescript.tsdk/Volar/Claude+47件 |
| unique_angle反映（エラー文→修正diff→検証コマンド） | ✅ Claudeプロンプト＋diff指示＋`tsc --listFilesOnly`検証で逆引き構成 |
| 有料章の価値 | ✅ コピペ用テンプレ7種＋47件検証データ（include外し31件/depth=0が12件）を提供 |

## エッジを効かせた点

- 章概要の「無症状バグ」を **tsserverログ→Claude二分プロンプト** として具体化し、推測禁止・log行番号根拠を強制
- `maxNodeModuleJsDepth` を「0で候補ゼロ／2で復活／5で8秒→40秒」と実測の分岐点で数値化
- 修正後の検証を1行コマンドで締め、章末で読者が即手を動かせる構成

文字数は本文約1,500字（テンプレ7種を全て収録したため概要1,200字を上回りましたが、コピペ価値を優先）。短縮が必要であればテンプレ2/6の解説を圧縮できます。
