---
title: "Claudeプロンプト設計：パイプラインに安定品質の記事を生成させる"
free: false
---

## Claudeプロンプト設計：パイプラインに安定品質の記事を生成させる

自動パイプラインで記事を量産するとき、最大の敵は「品質のブレ」だ。同じコードを実行しても、ある日はSEO最適化された700字の記事が出て、次の日は関係ない話題の200字が返ってくる。これを防ぐのがプロンプト設計の役割である。

---

## system / user ロールの分離が品質安定の土台

Claude APIには `system` と `user` という2種類のロールがある。役割を混在させると出力が不安定になる。

| ロール | 用途 |
|--------|------|
| `system` | 記事の性格・制約・口調を固定する |
| `user` | 今回のトピック・ターゲットKWを渡す |

```python
import anthropic

client = anthropic.Anthropic()

SYSTEM_PROMPT = """
あなたは副業・節約ジャンルの記事ライターです。
以下のルールを必ず守ってください。

- 文体: 常体（〜だ・〜である）
- 文字数: 600〜800字（厳守）
- 構成: リード文→本文3段落→まとめ
- ジャンル: 副業・在宅収入・節約に限定する
  （関係ないトピックが来た場合は「ジャンル外です」と返す）
- アフィリエイト文言: 本文末尾の「まとめ」段落の最後に必ず1文挿入すること
  例: 「詳しくはこちら→[A8アフィリリンク]」
"""

def generate_article(keyword: str, affiliate_link: str) -> str:
    user_prompt = f"""
以下のキーワードで記事を書いてください。

キーワード: {keyword}
アフィリエイトURL: {affiliate_link}
"""
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1200,
        system=SYSTEM_PROMPT,
        messages=[{"role": "user", "content": user_prompt}]
    )
    return response.content[0].text
```

---

## 落とし穴1：文字数制御は「範囲指定＋罰則」で固める

`「800字で書いて」` だけでは機能しない。モデルは文字数を正確に計算しない。効くのは**範囲指定＋逸脱時の指示**だ。

```
- 文字数: 600〜800字（厳守）
- 超過した場合は「まとめ」を削ってでも800字以内に収めること
- 不足の場合は本文第2段落を1文追加して補うこと
```

それでもズレる場合は、生成後にPythonで検証してリトライする。

```python
def generate_with_length_check(keyword: str, affiliate_link: str, 
                                 min_len=600, max_len=800, retries=2) -> str:
    for attempt in range(retries + 1):
        article = generate_article(keyword, affiliate_link)
        length = len(article.replace("\n", ""))
        if min_len <= length <= max_len:
            return article
        if attempt < retries:
            print(f"文字数NG ({length}字), リトライ {attempt+1}/{retries}")
    # 最終試行の結果をそのまま返す（パイプラインを止めない）
    return article
```

---

## 落とし穴2：ジャンルドリフトを防ぐ

パイプラインにキーワードを動的に渡すと、無関係なキーワードが混入したとき記事がジャンルから外れる。systemプロンプトに**明示的な拒否条件**を書いておくと、不良コンテンツをパイプライン外に弾ける。

```python
def is_valid_article(text: str) -> bool:
    return "ジャンル外です" not in text
```

ジャンル外判定が返ってきた行をログに残し、キーワードリストから除外するだけで品質が大幅に安定する。

---

## アフィリエイト文言の挿入パターン

URLをuser側に渡すことで、**記事ごとに異なるリンクを動的に差し込める**。systemで「末尾に1文」と構造を固定し、userでURLを渡す設計が最もトラブルが少ない。

```
# systemで固定する構造
- アフィリエイト文言: まとめ段落の末尾に必ず以下の形式で挿入
  「→ [サービス名]の詳細・申し込みはこちら（{affiliate_link}）」

# userで渡す変数
アフィリエイトURL: https://px.a8.net/svt/ejp?a8mat=xxxxx
```

systemに直接URLをハードコードすると、リンクを変えるたびにsystemプロンプト全体を書き換えることになる。変動値は必ずuserに分離するのがパイプライン設計の鉄則だ。

---

## まとめ：設計の3原則

1. **性格と制約はsystemに集中させる** — ジャンル・文体・文字数・構成はすべてsystem
2. **変動値はuserに渡す** — キーワード・URLなど記事ごとに変わるものはuser
3. **生成後に数値検証するコードを挟む** — プロンプトだけを信用せず、長さ・必須文言の存在をコードで確認する

この3原則を守るだけで、100本単位の量産でも品質のばらつきが劇的に減る。
