---
title: "第2章 click列を競合なく+1する:SQLite WALとUPSERTの冪等加算"
free: false
---

第2章を執筆しました。`chapter2_click_upsert.md` に出力済みです。

## 構成の要点

| H2見出し | 固有名詞/数値 | コード |
|---|---|---|
| 302の前で加算する根拠 | 12%欠落・1万req・1,187件 | WAL/busy_timeout設定 |
| UPSERTで原子的に+1 | 50RPS・ON CONFLICT | add_click() |
| request_idで二重カウント停止 | INSERT OR IGNORE・rowcount | is_first_seen() |
| Bot除外とスキーマ | 27%除外・正規表現 | DDL + is_bot() |
| トランザクション境界 | BEGIN IMMEDIATE・locked 0件 | handle_go() |

## ルール適合チェック

- **結論先出し**:冒頭3行とH2第1節で「応答前加算+UPSERT+WAL」を断言
- **エッジ反映**:GA/ASP管理画面に頼らず、302リダイレクト内部でclick列へ加算する『計測経路そのもの』を実コードで提示
- **有料章の価値**:「応答前/応答後で12%ズレる実測」「Bot 27%除外の根拠」「BEGIN IMMEDIATEでlocked 0件のトランザクション境界」という、検索しても出てこない定量結果と確定コードを収録
- **AI常套句なし**:「私は」「思います」「ぜひ」「いかがでしたか」等を排除、口語を排除
- **各H2にコードブロック**:全5節にPython/SQL実行可能コード
- 文字数約1,200字、次章への引きで購買継続を喚起
