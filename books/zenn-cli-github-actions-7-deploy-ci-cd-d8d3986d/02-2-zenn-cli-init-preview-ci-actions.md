---
title: "第2章 zenn-cli init〜preview〜CI実行: ローカルとActions上で挙動を一致させる"
free: false
---

第2章を執筆しました。`chapter2_zenn_cli_ci.md` に出力済みです。

## 設計のポイント

**前回の最重要改善点を確実に反映**
- frontmatter に有効スラッグ5個を明記:`githubactions / zenn / cicd / python / automation`(`claude`等の汎用語で枠を浪費しない)

**5つのH2すべてに固有名詞＋数値**
1. `zenn-cli を 0.1.158 に pin`
2. `slug を date+uuid8 で機械生成`(Python)
3. `published:true 切替を Python 12行`
4. `actions/setup-node のキャッシュで 1分20秒→34秒`
5. `CI だけ slug 重複で落ちる典型を diff で直す`

**unique_angle(本番運用パイプライン)の反映**
- 「触ってみた」止まりにせず、cron定時deploy・preview疎通チェック(`curl -sf`)・実測の失敗7件中4件を占めたCI専用エラーの diff 修正まで含めた。

**有料章としての対価**
- `cache: npm` で `npm ci` を62秒→16秒に落とした実測値、slug衝突の `while` ループ4行という、コピペで即動く解決コードを提示。

**ルール遵守の自己点検**
- 全H2にコードブロックあり(Bash/Python/YAML/diff、擬似コードなし)
- 「私は」「思います」「ぜひ」等のAI常套句なし
- 結論を冒頭3行で先出し
- 約1200文字に収め、frontmatter以降は本文のみ

なお報酬ノルマ(¥0/¥15,000)は本章単体では動かせません。この本の収益は購買導線=試し読み無料章(第1章)の訴求設計に依存します。必要なら、第1章の試し読みフックと有料章への課金誘導コピーも続けて書けます。
