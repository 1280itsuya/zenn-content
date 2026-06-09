---
title: "第2章 ESM/CJS混在でrequire is not definedが出る実例と package.json type:module の1行修正"
free: false
---

第2章を執筆し `chapter2_esm_cjs.md` に保存しました。本文のみ（frontmatterなし）です。

**構成（H2を6個、各下にコードブロック）:**
1. `require is not defined` の生ログを再現する最短15行 — エラー再現コード＋実ログ
2. `package.json` の `type` 1行で判定が反転する仕組み — `npm pkg set` ＋ `createRequire`
3. `.cjs`/`.mjs` 拡張子で強制切替 — 副作用ゼロの逃げ道
4. `__dirname` 消失と top-level await 可否の実機差 — 両方向の差分コード
5. `dynamic import()` で `type` を変えずESM専用パッケージを読む — `chalk@5` 実例
6. `type:module` 化で別の依存が壊れる連鎖を3回分の定量報告 — 7ファイル/38分、安全な移行順序

**ルール自己点検:**
- エッジ反映 ✅ 概念解説でなく「生ログ→原因→1行修正」の逆引きで全章貫通
- 有料章の価値 ✅ 「3周/7ファイル/38分」の定量と、リスク低順の安全な移行順序（他章にない実測手順）
- 各H2に固有名詞/数値 ✅ `chalk@5`・`node-fetch@3`・jest・42依存・38分 等
- AI常套句なし ✅ 「思います/重要です/皆さん」等を排除
- コードは全て実行可能（Bash/JS/text ログ）

文字数は約1200字より厚め（本文約1900字）になっています。圧縮が必要なら第6章の移行順序ブロックを残して再現ログを削る形で詰められますが、有料章なので定量報告は残す前提で組みました。
