---
title: "パイプライン設計の全体像：Claude APIで何が変わるのか"
free: true
---

## 手作業フローの限界

アフィリ記事を1本書くのに、従来は次のような工程が必要だった。

1. キーワードリサーチ（30分）
2. 競合記事の調査（20分）
3. 構成案の作成（15分）
4. 本文執筆（90〜120分）
5. 内部リンク・アフィリリンク挿入（15分）
6. CMS投稿・タグ設定（10分）

合計で1記事あたり約3時間。月20本で60時間。外注すれば1本3,000〜8,000円、月20本で最大16万円のコストがかかる。

Claude APIを使ったパイプラインでは、この工程を**ほぼ無人で**回せる。1記事あたりの生成コストは入出力トークン合計でおよそ0.01〜0.03ドル（claude-sonnet-4-6使用時）。月20本でも数十円台だ。

---

## パイプラインの全体アーキテクチャ

```
[キーワードリスト .csv]
        ↓
  keyword_picker.py         # 日付ローテーションで重複回避
        ↓
  outline_agent.py          # Claude API: H2/H3構成を生成
        ↓
  writer_agent.py           # Claude API: セクションごとに本文を生成
        ↓
  affiliate_injector.py     # 正規表現でアフィリリンクを注入
        ↓
  poster.py                 # WordPress REST API / Zenn API で投稿
        ↓
  report.py                 # 投稿URLをSlack/Discordへ通知
```

各コンポーネントは独立したPythonスクリプトで、`.env`ファイルの設定だけで有効・無効を切り替えられる。これがあとで「Zennだけ止めたい」「アフィリ注入をテストしたい」という場面で効いてくる。

---

## 必要コンポーネントと役割

| コンポーネント | 役割 | 主な依存 |
|---|---|---|
| `keyword_picker` | CSV から当日分KWを選出 | pandas |
| `outline_agent` | 見出し構成をJSON生成 | anthropic SDK |
| `writer_agent` | 本文をセクション単位で生成 | anthropic SDK |
| `affiliate_injector` | アフィリIDを本文に挿入 | re, dotenv |
| `poster` | CMS へ投稿 | requests |
| `report` | 結果通知 | httpx |

コードの骨格はこうなる。

```python
import anthropic

client = anthropic.Anthropic()  # ANTHROPIC_API_KEY を .env から自動読込

def generate_outline(keyword: str) -> dict:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=512,
        messages=[{
            "role": "user",
            "content": f"「{keyword}」に関するアフィリ記事の見出し構成をJSON形式で返してください。"
                       "キー: title, sections(リスト)"
        }]
    )
    import json
    return json.loads(response.content[0].text)
```

---

## よくある落とし穴

**1. タイトルと本文のテーマがずれる**

`outline_agent`が生成した見出しを`writer_agent`に渡す際、システムプロンプトで「あなたは{keyword}に関する記事を書いています」と毎セクション明示しないと、話題が脱線する。特に長文生成ではモデルがコンテキストを薄めやすい。

**2. アフィリリンク挿入で本文が壊れる**

```python
# 危険: 単純置換は部分一致で別の単語を壊す
text = text.replace("クレジットカード", f'<a href="{url}">クレジットカード</a>')

# 安全: 単語境界を考慮した置換
import re
pattern = r'(?<![ア-ン一-龥a-zA-Z])クレジットカード(?![ア-ン一-龥a-zA-Z])'
text = re.sub(pattern, f'<a href="{url}">クレジットカード</a>', text, count=1)
```

`count=1`で最初の出現のみ置換するのが基本。複数回差し込むとスパム判定のリスクが上がる。

**3. APIエラー時に無音でスキップされる**

ネットワーク障害やレート超過でセクションが空になっても、パイプラインが次の工程に進んでしまうケースがある。各エージェントの出力に対して「最低文字数チェック」を挟み、空または短すぎる場合は例外を投げてパイプラインを止める設計にすること。

---

## まとめ：このパイプラインで変わること

- 1記事の人的工数：3時間 → 5分（確認・投稿承認のみ）
- 月間コスト：外注16万円 → API代数十円
- スケール上限：体力依存 → キーワードCSVの行数だけ

次章では、このアーキテクチャの入り口である`keyword_picker`の実装と、重複投稿を防ぐローテーション戦略を詳しく解説する。
