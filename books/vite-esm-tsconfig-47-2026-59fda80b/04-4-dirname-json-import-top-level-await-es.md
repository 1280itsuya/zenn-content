---
title: "第4章: __dirname消失・JSON import・top-level awaitなどESM固有エラーの最小修正パターン"
free: false
---

第4章を執筆しました。`chapter4_esm_errors.md` に出力済みです。

**構成（エラー文字列を見出しにした逆引き／全4 H2・各コードブロック付き）**

1. `__dirname is not defined in ES module scope` → `import.meta.url` + `fileURLToPath` 復元（Before/After最小diff）
2. `Module needs an import attribute of type "json"` → Node 22の`with`/旧`assert`のバージョン別1語差＋`resolveJsonModule`必須
3. `Top-level await is not available` → `TS1378`=target、`TS1432`=module とエラー番号で直す側を即断
4. CJS interop → `esModuleInterop` / `allowSyntheticDefaultImports` の**実害対応表**（症状エラー→切るフラグ）

**有料章の価値**: 「`allowSyntheticDefaultImports`単独ONはコンパイルだけ通り実行時に壊れる」という落とし穴と、TS1378/1432・TS1192のエラー番号→修正箇所の対応表を提供。

**自己点検**
- コードブロック: 各見出しに最低1つ（計5、表含む）✓
- AI常套句（私は/思います/ぜひ/皆さん 等）: なし ✓
- 各見出しに固有名詞・数値: Node 22/Vite 6/ES2022/TS1378 等 ✓
- unique_angle（エラー文字列＝見出しの逆引き）: 全H2がエラー文字列起点 ✓
- 冒頭にゴール1文先出し ✓

なおこの章はテック教材で、月次ノルマ対象のアフィリエイト導線（ふるさと納税・格安SIM等）とはテーマが噛み合わないため、本文への無関係なアフィリ挿入はしていません（読者の信頼を損なうため）。導線は別途、集客系記事側で効かせる方針が妥当です。
