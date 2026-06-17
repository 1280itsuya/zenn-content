---
title: "第2章：Claude APIセットアップとアフィリ記事生成に最適なプロンプト設計"
free: false
---

## Claude APIキーの取得と初期設定

まず [console.anthropic.com](https://console.anthropic.com) にアクセスし、アカウントを作成する。ダッシュボードの **API Keys** から新しいキーを発行したら、プロジェクトルートの `.env` に保存する。

```env
ANTHROPIC_API_KEY=sk-ant-api03-xxxxxxxxxxxx
```

コード側では `python-dotenv` で読み込む。**キーをソースコードに直書きしない**のは鉄則だ。Gitに誤ってpushするとAnthropicが自動検知して即失効させる。

```python
from dotenv import load_dotenv
import anthropic, os

load_dotenv()
client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])
```

---

## アフィリ記事生成に最適なモデル選択

現時点では **claude-sonnet-4-6** が品質・速度・コストのバランスで最良だ。1記事（約2,000字）あたりの入出力トークンは概算で入力800・出力1,200トークン。料金は$0.003前後に収まる。

| モデル | 用途 | 1記事コスト目安 |
|--------|------|-----------------|
| claude-haiku-4-5 | 下書き・アウトライン | ~$0.0005 |
| claude-sonnet-4-6 | 本文生成（推奨） | ~$0.003 |
| claude-opus-4-8 | 最高品質・単価高め | ~$0.03 |

量産パイプラインでは **Sonnet で本文を生成し、Haiku でタイトルとメタ概要を生成** する二段構えが費用対効果が高い。

---

## system プロンプト：キャラクターと制約を固定する

`system` パラメータはリクエストごとに変わらない「ライターの人格」を定義する場所だ。ここが曖昧だと記事のトーンがブレる。

```python
SYSTEM_PROMPT = """
あなたは日本語の副業・節約系アフィリエイトブログ専門ライターです。
以下のルールを必ず守ってください：

- 読者は20〜40代の会社員。専門用語は使わず平易な言葉で書く
- 文体は「です・ます」調。親しみやすく、しかし信頼感のある語り口
- 商品の誇大表現・断定的な収益保証は書かない（景品表示法に抵触するため）
- 必ず具体的な数字・手順・体験談的な描写を入れる
- アフィリリンク設置箇所は「[AFFILIATE_LINK]」プレースホルダで示す
"""
```

**落とし穴①**：`system` を省略すると Claude はデフォルトの汎用アシスタントとして振る舞い、アフィリ記事特有の「背中を押す結論」や「比較表」を自発的に書かなくなる。

---

## user プロンプト：テンプレ変数で量産を制御する

`user` プロンプトは記事ごとに差し替える部分だ。**f文字列でニッチとキーワードを注入**するだけで、100本のバリエーションを生成できる。

```python
def build_user_prompt(niche: str, keyword: str, product_name: str) -> str:
    return f"""
以下の条件でアフィリエイト記事本文を日本語2,000字で書いてください。

## 記事条件
- ニッチ: {niche}
- メインキーワード: {keyword}（タイトル・H2・本文3回以上自然に含める）
- 紹介商品: {product_name}
- 構成: 導入(課題提起) → 解決策(商品紹介) → 使い方 → メリット/デメリット → まとめ(CTA)

## 出力形式
Markdown。見出しは##と###のみ使用。
アフィリリンクは [AFFILIATE_LINK] で示すこと。
"""
```

**落とし穴②**：キーワードを「自然に含める」と指示しないと、Claude は冒頭だけに詰め込んでSEO的に不自然な文章を生成することがある。

---

## API呼び出しの実装例

```python
def generate_article(niche: str, keyword: str, product_name: str) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        system=SYSTEM_PROMPT,
        messages=[
            {"role": "user", "content": build_user_prompt(niche, keyword, product_name)}
        ]
    )
    return response.content[0].text

# 使用例
article = generate_article(
    niche="クレジットカード比較",
    keyword="楽天カード 審査 通らない",
    product_name="イオンカードセレクト"
)
```

`max_tokens=2048` は2,000字の日本語本文に対して十分な余裕だ。これを下回ると文章が途中で切れる。**出力が `stop_reason: "max_tokens"` で終わっていないか**、毎回 `response.stop_reason` を確認する習慣をつけよう。

---

## プロンプト品質チェックの指標

生成後に以下を自動検証するとゴミ記事の量産を防げる。

```python
def validate_article(text: str, keyword: str) -> bool:
    word_count = len(text.replace(" ", ""))
    keyword_count = text.count(keyword)
    has_affiliate_placeholder = "[AFFILIATE_LINK]" in text

    return (
        800 <= word_count <= 3000
        and keyword_count >= 2
        and has_affiliate_placeholder
    )
```

この3条件（文字数・キーワード出現・プレースホルダ存在）を通過しない記事は再生成キューに戻す。次章では、この再生成ロジックを含むパイプライン全体の自動化に進む。
