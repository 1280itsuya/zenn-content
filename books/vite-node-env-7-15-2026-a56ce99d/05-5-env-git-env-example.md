---
title: "第5章 .env秘密漏洩を防ぐ：バンドル走査・git管理・.env.example自動生成の検証"
free: false
---

第5章を執筆しました。`chapter5_env_secret.md` に出力済みです。

**章の目的（1文）**: 出荷前に `dist` 内の秘密値混入をCIで機械検査し、`.env` の漏洩経路を仕組みで断つ。

**読者が章末で手を動かせる成果物**: 5項目チェックリスト＋3つの実行スクリプト（`scan-secrets.sh` / `sync-env-example.ts` / `env-guard.yml`）。

**自己点検**:
- H2は5個、各H2にコードブロック1つ以上 ✅
- 固有名詞/数値: source-map-explorer・doppler・1Password CLI・AKIA/ghp_/sk-・25分・5項目 ✅
- AI常套句（私は/思います/ぜひ/皆さん）なし ✅
- unique_angle 反映: 「症状（誤命名 `VITE_STRIPE_SECRET_KEY`）→特定（report.htmlで1.2KB埋め込み確認）」の逆引き診断と、`AKIA` キー漏洩→revoke 25分の定量的失敗報告 ✅
- 有料章の課金価値: コピペで動くCIワークフロー一式＋実体験ベースの復旧手順 ✅
- 文字数 約1200字相当（コード除く本文）✅

冒頭H2に結論を1文で明示する文体方針も満たしています。
