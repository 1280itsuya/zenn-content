---
title: "品質ゲートとレビュー自動化：LLM-as-judgeでスパム記事を防ぐ"
free: false
---

## LLM-as-judgeとは何か

生成記事をそのまま公開すると、「タイトルと本文がずれている」「同じ表現が3回繰り返される」「300字で終わる薄いコンテンツ」が混入する。人手でチェックするとパイプラインの意味がない。解決策は **Claude自身を審査員（judge）として使う**ことだ。

生成エージェントとは別のプロンプトで「この記事は品質基準を満たしているか」と問い、スコアと理由を返させる。合格なら投稿、不合格なら再生成にかける。

---

## judgeエージェントの実装

```python
import anthropic, json

client = anthropic.Anthropic()

JUDGE_PROMPT = """
あなたは記事品質審査員です。以下の記事を評価し、JSONで返してください。

## 評価基準
- theme_ok: タイトルと本文の主題が一致している (true/false)
- duplicate_ratio: 重複表現の割合 (0.0〜1.0)
- word_count: 本文の文字数
- verdict: "pass" or "fail"
- reason: failの場合のみ理由を1行で

## 記事
タイトル: {title}
本文:
{body}

JSONのみ返すこと。説明不要。
"""

def judge_article(title: str, body: str) -> dict:
    resp = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=300,
        messages=[{
            "role": "user",
            "content": JUDGE_PROMPT.format(title=title, body=body)
        }]
    )
    return json.loads(resp.content[0].text)
```

評価結果の例：

```json
{
  "theme_ok": false,
  "duplicate_ratio": 0.12,
  "word_count": 820,
  "verdict": "fail",
  "reason": "タイトルは『Python自動化』だが本文は『ChatGPT活用術』にすり替わっている"
}
```

---

## 再生成ループへの組み込み

```python
MAX_RETRY = 3

def generate_with_gate(title: str, theme: str) -> str | None:
    for attempt in range(MAX_RETRY):
        body = generate_article(title, theme)   # 生成エージェント
        result = judge_article(title, body)

        if result["verdict"] == "pass":
            return body

        # 再生成時に失敗理由をフィードバック
        theme = f"{theme}\n\n【前回の失敗理由】{result['reason']}"

    return None   # 3回失敗したらスキップ
```

ポイントは**失敗理由を次の生成プロンプトに混ぜる**こと。ただ「もう一度」と言うだけでは同じ失敗を繰り返す。

---

## よくある落とし穴

**judgeも幻覚する**。`theme_ok: true`を返しても本文を読むと明らかにずれているケースがある。対策は評価基準を具体化することだ。「主題が一致」では曖昧なので「タイトルのキーワードXが本文中に3回以上登場し、かつ本文の結論がXに関する内容であること」のように数値・出現条件で縛る。

**スコアのインフレ**も起きやすい。Claude は同一セッション内で生成した文章を甘く採点する傾向がある。生成と審査で **別の `client.messages.create` 呼び出し**にするだけでなく、system プロンプトで「厳しく採点し、疑わしければfailにせよ」と明示的に指示する。

**コスト管理**を忘れない。judge は入出力トークンが小さいので `claude-haiku-4-5-20251001` で十分なケースが多い。生成に Sonnet、審査に Haiku と使い分けると API コストを半減できる。

---

## 実運用での判断基準

| チェック項目 | 閾値の目安 | 対処 |
|---|---|---|
| `theme_ok` | false | 即再生成 |
| `duplicate_ratio` | 0.15以上 | 再生成 |
| `word_count` | 500字未満 | 再生成 |
| 3回失敗 | — | スキップしてログに記録 |

スキップした記事はログに残し、後日プロンプトのチューニング素材にする。「なぜjudgeを通れなかったか」の傾向を分析すると、生成プロンプト側の構造的な欠陥が見えてくる。
