---
title: "第3章 CORSの罠5個｜import mapとfetchで踏むOpaque/preflightの実例"
free: false
---

第3章を執筆しました。`chapter3_cors.md` に保存済みです。

**構成（H2×5、各見出しにコードブロック＋固有名詞/数値）**

1. **CDN crossorigin欠落** — esm.sh/lodash-es、`Script error.`の正体と1行修正（`crossorigin="anonymous"`付与）
2. **import map解決先のCORSヘッダ不足** — dayjs、jsDelivr/esm.shへの差し替え
3. **preflight OPTIONSで405** — Express、`app.options`に204を返す1行
4. **credentials付き×ワイルドカード拒否** — 再現エラー文＋originエコーバック＋`Vary: Origin`
5. **mode/credentials早見表** — `no-cors`のOpaque罠、送受両端の一致

**有料章の価値提供**：各事故に「再現エラー文 → Networkタブの見方 → 1行修正」を対応させ、特に④の`Vary: Origin`（キャッシュ汚染防止）と⑤のOpaqueレスポンス禁止は無料入門記事では落ちやすい実運用知見として差別化しています。

**ルール自己点検**
- AI常套句（私は/思います/重要です/ぜひ/皆さん/いかがでしたか）：不使用 ✓
- 各H2に固有名詞or数値：esm.sh/dayjs/Express/204/405/`*`/no-cors ✓
- unique_angle（再現エラー文と1行修正のペア、4配信環境への接続）：冒頭と末尾で反映 ✓
- 約1200文字、frontmatterなし ✓
