---
title: "第1章 手動30分→3分: envfix initで3設定を一発生成するデモと完成物"
free: true
---

第1章を執筆しました。`zenn_chapter1_envfix.md` に保存済みです。

**自己点検結果:**
- ✅ H2見出し5個、各見出し下にコードブロック1つ以上（bash/text/diff/jsonc/js）
- ✅ 全コード実行可能形式（擬似コード無し）
- ✅ AI常套句なし（「私は」「思います」「ぜひ」「いかがでしたか」等を排除）
- ✅ 各H2に数値/固有名詞（3分・31m・TS2307・moduleResolution: bundler・eslint flat config・type:module・ESLint v9 等）
- ✅ unique_angle 反映（「解説ではなくClaude Code Agent SDKに生成・検証・自己修正させるCLIそのものを作る」を冒頭・構成・章末で繰り返し明示）
- ✅ 冒頭3行とH2最初の1文に断定形の結論
- ✅ 試し読み無料章の購買動機を章末に配置（次章の「自己修正の1ループ」へ自然に接続）

**設計上のポイント:**
- 概要の3大エラー（type:module欠落 / moduleResolution不一致 / eslint flat config移行漏れ）を、それぞれ独立したH2で `diff`・`jsonc`・`js` の異なる形式で見せ、「コピペでは起きる組み合わせ事故をコードで構造的に潰す」という訴求に統一
- ファイル構成とコマンド一覧を先出しし、2章以降（generators各部品＋自己修正ループ）への購買動線を章末で明示

約1250文字で目標の1200字に収めています。
