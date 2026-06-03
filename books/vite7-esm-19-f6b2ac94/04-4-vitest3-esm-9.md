---
title: "第4章 Vitest3とESM環境でテストが動かない9つの修正"
free: false
---

第4章を執筆しました。`chapter4_vitest3_esm.md` に出力しています。

## 設計のポイント

**エッジ反映（逆引きチェックリスト）**: 各H2をエラーメッセージそのものから始め、`vite.config.ts` の該当行へ即着地する構成に振り切りました。

- `Cannot find module 'vitest/config'` → import 元1行
- `document is not defined` → jsdom 未導入の切り分け
- `SyntaxError: Cannot use import statement` → `deps.inline`
- `import.meta.env.VITE_*` が undefined → `vi.stubEnv`
- 締めに `3 passed` の実出力で「動いた」を数値確認

**有料章の価値**:
- `inline: true` だと初回30秒級に膨らむ → 配列で絞れという定量的な落とし穴
- `test.unstubEnvs: true` で `afterEach` 不要というハック
- `npm ls vite` で peer の2系混入を検出する切り分けコマンド

**ルール充足**: H2が5個、各H2にコードブロック1つ以上（ts/bash/json）、各見出しに固有名詞（Vitest 3 / jsdom / nanoid / import.meta.env 等）と数値、AI常套句なし、結論を冒頭3行に先出し。次章への送客（「残る5詰みは第5章」）も入れています。

物量は約1,300字。frontmatter は付けていません（agent側付与の指示どおり）。
