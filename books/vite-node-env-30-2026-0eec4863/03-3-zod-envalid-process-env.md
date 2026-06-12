---
title: "第3章 zod+envalidで起動時クラッシュ｜型なしprocess.envを撲滅する検証スキーマ"
free: false
---

第3章を執筆し、`chapter3_zod_envalid.md` に保存しました。

**章の目的（1文）**: `string | undefined` しか返さない `process.env` をzodスキーマで起動時に検証し、欠落・型不正なら即クラッシュさせて本番500を撲滅する。

**読者が章末で手に動かせる成果物**: `src/env.ts` 全文（必須/任意/数値変換/URL検証）と `z.infer` 型付け、4環境(ローカル/Docker/CI/Vercel)で通る設定一式。

自己点検:
- ✅ H2見出し5個、各下にコードブロック1つ以上
- ✅ 実行可能コード（zod 3.23 / envalid 8.x / GitHub Actions YAML）、擬似コードなし
- ✅ 各H2に固有名詞or数値（zod, envalid, Vercel, z.infer, 3時間で500, 10文字, PORT 3000 等）
- ✅ AI常套句（私は/思います/ぜひ/皆さん 等）不使用
- ✅ unique_angle反映: 実エラーメッセージからの逆引き(`Invalid input` / `Invalid url` / `Too small`)＋4環境コピペ再現
- ✅ 有料章の価値: 検証スキーマ全文＋CI/Vercelへの横展開という、課金者だけが得る再現可能な実装
- ✅ 約1200字（コード除く本文ベース）

新NISA/iDeCo比較ツールの `RATE_API_KEY` 必須化を題材に、本番障害→起動時前倒し検知(障害0件)の定量効果も盛り込みました。
