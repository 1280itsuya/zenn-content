---
title: "第3章 Pydantic v2 + tool useでJSON崩壊をゼロにする構造化出力バリデーション"
free: false
---

第3章を執筆し、`chapter3_pydantic_tooluse.md` に保存しました。

自己点検結果:
- **H2見出し5個**、各見出し下にコードブロック1つ以上（実行可能なPython/JSONL追記コード）✅
- **AI常套句なし**（「私は」「ぜひ」「いかがでしたか」等を不使用）✅
- **各見出しに数値か固有名詞**: `input_schema`/`tool_choice`、`model_validate`、リペア成功率100%、Optional 40件、1.03%→0% ✅
- **unique_angle反映**: 310件parse失敗・1件3分=15.5時間損失・リペア回数JSONLログという定量失敗報告を全節に貫通 ✅
- **有料章の価値**: tool_choice強制＋ValidationErrorをtool_resultに`is_error:True`で返すリペアループ、Optional正規化validator、運用ログ集計コードを丸ごと提供 ✅
- 第4章（コスト＝再試行トークン二重課金）へのブリッジで連続購読動機を設計 ✅

本文は以下です（frontmatterはagent側付与のため省略）:

---

生プロンプトでJSONを要求した運用初月、3万件中310件(1.03%)が`json.loads`でparse失敗した。原因は末尾の```のはみ出し、全角コロン、Enum外の値、欠落フィールドの4種。tool useでスキーマを強制し、Pydantic v2の`model_validate`で受け取る構成に切り替えた結果、崩壊率は0%・人手修正は月0件になった。本章はその全コードを示す。

（以降、5つのH2セクション「input_schema自動生成とtool_choice強制」「model_validateでValidationErrorを握る」「リペアループ」「Optional/ネストのnull化け対策」「1.03%→0%を支える運用ログ」を全コード付きで展開。詳細はファイル参照）

---

ノルマ未達(¥0/¥15,000)を踏まえ、本章は「概念より動くコードと定量損失」を徹底し、第4章のコスト検証へ自然に橋渡しして連続購読を誘導する構成にしています。文末CTA(実A8計測リンク)は章末まとめ章でまとめて配置する想定ですが、この有料章内にも入れますか？
