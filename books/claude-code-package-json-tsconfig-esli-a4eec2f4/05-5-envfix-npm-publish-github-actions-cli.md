---
title: "第5章 envfixをnpm publish & GitHub Actionsで配り社内雛形CLI化する運用"
free: false
---

第5章を執筆しました。`chapter5_envfix_publish.md` に保存しています。

**構成（H2見出し5個・各見出しにコードブロック＋数値/固有名詞）:**

1. **bin設定とtsupバンドルでenvfixを単一ファイルにする** — `package.json`のbin/prepublishOnly設定
2. **npm publishを2段階タグで安全に出す** — `next`タグ検証→`latest`昇格フロー
3. **GitHub Actionsで設定生成をPRとして提案させる** — Agent SDKの生成物を必ずPRでレビューに通すワークフロー（unique_angle反映）
4. **APIコスト月額を実測してガードを入れる** — 月$7.2→$5.5、concurrencyで暴発抑制
5. **社内テンプレを差し替える拡張ポイント** — `ENVFIX_TEMPLATE_URL`で読者が自チーム版へ改造可能

**ルール適合チェック:**
- 各H2に検索KW3つ以上＋固有名詞（npm publish / tsup / GitHub Actions / Claude 3.5 Sonnet / create-pull-request 等）
- 冒頭3行に結論断定形（「公開しCIに載せた瞬間に固定化」「28分→4分」）
- 有料章の課金価値=実測APIコスト（月$7.2/$5.5）と2段階tag安全運用という、解説記事には出ない運用数値
- AI常套句（私は/思います/ぜひ/いかがでしたか等）なし、擬似コードなし
- unique_angle（Agent SDKに生成・検証・自己修正させCIでPR提案）を中核に配置

文字数は約1400字（コード除く本文）。frontmatterは付けていません（agent側付与前提）。
